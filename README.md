# Lambda S3 Processor Service

Module này tạo Lambda functions được trigger bởi **EventBridge Scheduler** theo lịch định kỳ để xử lý file `manifest.json` từ S3 và đồng bộ lineage với OpenMetadata. Lambda functions chạy trong VPC với subnet và security group được cấu hình.

## 📋 Features

- ✅ **EventBridge Scheduler**: Trigger Lambda theo lịch định kỳ (rate hoặc cron expression)
- ✅ **VPC-enabled Lambda**: Chạy trong VPC với private subnets và security groups
- ✅ **OpenMetadata Integration**: Đồng bộ lineage từ dbt manifest.json vào OpenMetadata
- ✅ **Hub-Spoke Lineage**: Tự động tạo lineage giữa các bảng có tên giống nhau trong hub_database và spoke_database
- ✅ **AWS Secrets Manager**: Lấy database credentials và API tokens từ Secrets Manager
- ✅ **PostgreSQL Connection**: Kết nối trực tiếp với OpenMetadata database để query và lưu state
- ✅ **Docker Build Support**: Build Lambda package với native dependencies (psycopg2)
- ✅ **Auto Naming Convention**: `{env}-{last4digits}-{region_short}-{name}`
- ✅ **CloudWatch Logs**: Logs với retention period configurable
- ✅ **Custom IAM Policies**: Permissions cho S3, Secrets Manager, CloudWatch, VPC

## 🏗️ Kiến Trúc Triển Khai

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS EventBridge                             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Scheduled Rule: rate(1 hour)                                │   │
│  │  Target: Lambda Function                                     │   │
│  └──────────────────────────┬───────────────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────────────┘
                              │ (Trigger every hour)
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS Lambda (in VPC)                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Function: manifest-processor                                │   │
│  │  Runtime: Python 3.11                                        │   │
│  │  Timeout: 300s │ Memory: 512MB                               │   │
│  │                                                              │   │
│  │  Environment Variables:                                      │   │
│  │  - MANIFEST_BUCKET: S3 bucket name                           │   │
│  │  - MANIFEST_KEY: manifest.json                               │   │
│  │  - OM_DB_SECRET: Secrets Manager secret name                 │   │
│  │  - OM_TOKEN_SECRET: API token secret name                    │   │
│  │  - HUB_DATABASE: hub database name                           │   │
│  │  - SPOKE_DATABASE: spoke database name                       │   │
│  └────────┬─────────────┬──────────────┬────────────────────────┘   │
│           │             │              │                            │
└───────────┼─────────────┼──────────────┼────────────────────────────┘
            │             │              │
            │             │              └──────────────┐
            │             │                             │
            ▼             ▼                             ▼
    ┌──────────────┐ ┌──────────────────┐   ┌──────────────────────┐
    │   AWS S3     │ │ AWS Secrets Mgr  │   │  CloudWatch Logs     │
    │              │ │                  │   │                      │
    │ manifest.json│ │ - DB Credentials │   │  /aws/lambda/        │
    │              │ │ - API Tokens     │   │  {function-name}     │
    └──────────────┘ └──────────────────┘   └──────────────────────┘
            │                  │
            │                  │
            ▼                  ▼
    ┌───────────────────────────────────────────┐
    │    OpenMetadata PostgreSQL Database       │
    │                                           │
    │  Tables:                                  │
    │  - table_entity (OM metadata)             │
    │  - om_lineage_state (lineage tracking)    │
    │                                           │
    │  Queries:                                 │
    │  1. Find matching tables (hub ↔ spoke)    │
    │  2. Save lineage state                    │
    └───────────────────────────────────────────┘
            │
            │ (REST API Calls)
            ▼
    ┌───────────────────────────────────────────┐
    │    OpenMetadata REST API                  │
    │                                           │
    │  PUT /v1/lineage                          │
    │  - Create lineage edges                   │
    │                                           │
    │  DELETE /v1/lineage/{from}/to/{to}        │
    │  - Delete outdated lineage                │
    └───────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      Processing Flow                                │
├─────────────────────────────────────────────────────────────────────┤
│  1. EventBridge triggers Lambda every hour                          │
│  2. Lambda reads MANIFEST_BUCKET and MANIFEST_KEY from env vars     │
│  3. Download manifest.json from S3                                  │
│  4. Get DB credentials and API token from Secrets Manager           │
│  5. Connect to OpenMetadata PostgreSQL database                     │
│  6. Initialize om_lineage_state table if not exists                 │
│  7. Process manifest lineage (dbt models):                          │
│     - Parse manifest.json for table dependencies                    │
│     - Compare with existing lineage in om_lineage_state             │
│     - Delete outdated lineage via API                               │
│     - Create new lineage via API                                    │
│     - Update om_lineage_state table                                 │
│  8. Create hub-spoke lineage (cross-database):                      │
│     - Query table_entity for matching table names                   │
│     - Filter hub_database and spoke_database                        │
│     - Create lineage spoke → hub via API                            │
│     - Save to om_lineage_state with ON CONFLICT DO NOTHING          │
│  9. Close connections and return success                            │
└─────────────────────────────────────────────────────────────────────┘
```

## 📁 Structure

```
services/lambda-s3-processor/
├── main.tf                           # Main service configuration
├── variables.tf                      # Input variables
├── outputs.tf                        # Output values
├── versions.tf                       # Provider configuration
├── terraform.tfvars                  # Configuration values
├── build_lambda.sh                   # Script to build Lambda packages
├── lambda/                           # Lambda function code
│   └── manifest-processor/
│       ├── index.py                  # Lambda handler
│       └── requirements.txt          # Python dependencies (optional)
└── README.md                         # This file

modules/layer/lambda-s3-processor/
├── main.tf                           # Layer module implementation
├── variables.tf                      # Layer variables
└── outputs.tf                        # Layer outputs
```

## 🚀 Usage

### 1. Configure terraform.tfvars

File `terraform.tfvars` chứa tất cả các cấu hình cho module. Dưới đây là giải thích chi tiết từng parameter:

#### 📌 Common Configuration

```hcl
stage = "dev"
```
- **stage**: Environment name (dev, staging, prod)
  - Dùng để tạo naming convention: `{stage}-{account_id}-{region}-{name}`
  - Ví dụ: `dev-7939-apse1-manifest-processor`

```hcl
tags = {
  Project     = "Lambda S3 Processor"
  Environment = "dev"
  ManagedBy   = "Terraform"
  Purpose     = "Process manifest.json changes from S3"
}
```
- **tags**: AWS resource tags cho tracking và cost allocation
  - `Project`: Tên project để nhóm resources
  - `Environment`: Environment name (dev, prod, etc.)
  - `ManagedBy`: Tool quản lý infrastructure (Terraform)
  - `Purpose`: Mục đích của resources

#### 🔐 IAM Role Configuration

```hcl
iam_role_name = "lambda-s3-processor"
```
- **iam_role_name**: Tên IAM role cho Lambda
  - Sẽ được thêm prefix: `{env}-{last4digits}-{region_short}-{iam_role_name}-role`
  - Ví dụ: `dev-7939-apse1-lambda-s3-processor-role`
  - Role này có trust relationship với `lambda.amazonaws.com`

```hcl
role_policy_arns = []
```
- **role_policy_arns**: List các AWS managed policy ARNs thêm vào role
  - Ví dụ: `["arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"]`
  - Để trống `[]` nếu chỉ dùng custom policies

```hcl
s3_bucket_arns = [
  "arn:aws:s3:::langfuse-ap-southeast-1-302010997939",
  # Add more bucket ARNs as needed
]
```
- **s3_bucket_arns**: List S3 bucket ARNs mà Lambda cần access
  - Custom IAM policy sẽ grant `s3:GetObject`, `s3:ListBucket` cho các buckets này
  - Format: `arn:aws:s3:::bucket-name`
  - **Lưu ý**: Policy tự động thêm `/*` suffix cho object-level permissions

#### 🌐 VPC Configuration

```hcl
vpc_subnet_ids = [
  "subnet-00110fb4eb114fba2",
  "subnet-08bbb6ff0964e4e39"
]
```
- **vpc_subnet_ids**: List subnet IDs mà Lambda sẽ chạy trong đó
  - **Bắt buộc**: Phải là **private subnets** với route đến NAT Gateway
  - Lambda tạo ENI (Elastic Network Interface) trong các subnets này
  - Cần ít nhất 1 subnet, khuyến nghị 2+ subnets cho high availability
  - Subnets phải có đủ IP addresses (recommended: /24 hoặc lớn hơn)
  - **Lưu ý**: Lambda cần NAT Gateway để gọi AWS APIs (S3, Secrets Manager, etc.)

```hcl
vpc_security_group_ids = [
  "sg-066d19cbab65cd84a"
]
```
- **vpc_security_group_ids**: List security group IDs áp dụng cho Lambda ENI
  - **Outbound rules cần thiết**:
    - Port 443 (HTTPS) → 0.0.0.0/0 cho AWS APIs (S3, Secrets Manager)
    - Port 5432 (PostgreSQL) → OpenMetadata database security group
  - **Inbound rules**: Thường không cần (trừ khi Lambda expose service)
  - Lambda sẽ áp dụng tất cả security groups trong list

#### ⚡ Lambda Functions Configuration

```hcl
lambda_functions = {
  "manifest-processor" = {
    function_name   = "manifest-processor"
    description     = "Lambda function to process manifest.json changes from S3"
    handler         = "index.lambda_handler"
    runtime         = "python3.11"
    timeout         = 300
    memory_size     = 512
    filename        = "lambda/manifest-processor.zip"
    
    environment = {
      variables = {
        LOG_LEVEL         = "INFO"
        MAIN_URL          = "https://open-metadata-eks/api"
        OM_DB_SECRET      = "prod/om-db"
        OM_TOKEN_SECRET   = "prod/om-token"
        MANIFEST_BUCKET   = "langfuse-ap-southeast-1-302010997939"
        MANIFEST_KEY      = "manifest.json"
        HUB_DATABASE      = "hub_database_name"
        SPOKE_DATABASE    = "spoke_database_name"
      }
    }
    
    logs_retention_days = 7
  }
}
```

**Giải thích từng parameter:**

- **function_name**: Tên Lambda function
  - Sẽ được thêm prefix: `{env}-{last4digits}-{region_short}-{function_name}`
  - Ví dụ: `dev-7939-apse1-manifest-processor`

- **description**: Mô tả function (hiển thị trong AWS Console)

- **handler**: Entry point của Lambda function
  - Format: `{file_name}.{function_name}`
  - `index.lambda_handler` → file `index.py`, function `lambda_handler(event, context)`

- **runtime**: Lambda runtime environment
  - Supported: `python3.11`, `python3.10`, `python3.9`, `nodejs20.x`, etc.
  - **Lưu ý**: Dùng `python3.11` cho psycopg2 compatibility

- **timeout**: Thời gian tối đa function chạy (seconds)
  - Min: 1s, Max: 900s (15 minutes)
  - `300` = 5 minutes (đủ cho processing manifest lớn)

- **memory_size**: RAM allocated cho function (MB)
  - Min: 128 MB, Max: 10,240 MB
  - `512` MB phù hợp cho processing manifest + database operations
  - **Lưu ý**: CPU power tỷ lệ thuận với memory allocation

- **filename**: Path đến Lambda deployment package (ZIP file)
  - Relative path từ service directory
  - Build bằng script: `./build_lambda.sh --docker`

**Environment Variables:**

- **LOG_LEVEL**: Logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`)
  - Mặc định: `INFO` cho production
  - Dùng `DEBUG` khi troubleshooting

- **MAIN_URL**: OpenMetadata REST API base URL
  - Format: `https://your-openmetadata-host/api`
  - Dùng để gọi lineage APIs: `PUT /v1/lineage`, `DELETE /v1/lineage/{from}/to/{to}`

- **OM_DB_SECRET**: AWS Secrets Manager secret name chứa OpenMetadata DB credentials
  - Secret format JSON:
    ```json
    {
      "host": "db.example.com",
      "port": 5432,
      "database": "openmetadata_db",
      "username": "om_user",
      "password": "secure_password"
    }
    ```
  - Lambda dùng boto3 để get secret value

- **OM_TOKEN_SECRET**: AWS Secrets Manager secret name chứa OpenMetadata API token
  - Secret format: Plain text JWT token
  - Dùng làm Bearer token trong API requests: `Authorization: Bearer {token}`

- **MANIFEST_BUCKET**: S3 bucket name chứa manifest.json
  - **Lưu ý**: Chỉ bucket name, không phải ARN
  - Ví dụ: `langfuse-ap-southeast-1-302010997939`
  - Lambda đọc từ env var thay vì EventBridge input

- **MANIFEST_KEY**: S3 object key của manifest file
  - Thường là: `manifest.json` hoặc `path/to/manifest.json`
  - Lambda kết hợp với MANIFEST_BUCKET để download: `s3://{bucket}/{key}`

- **HUB_DATABASE**: Tên database trong OpenMetadata chứa hub tables
  - Dùng để filter trong SQL query: `WHERE hub.json->'database'->>'name' = 'hub_database_name'`
  - Lambda tìm tables trong database này để tạo lineage với spoke

- **SPOKE_DATABASE**: Tên database trong OpenMetadata chứa spoke tables
  - Dùng để filter trong SQL query: `WHERE spoke.json->'database'->>'name' = 'spoke_database_name'`
  - Lambda tìm tables trong database này để tạo lineage với hub

- **logs_retention_days**: Số ngày giữ CloudWatch Logs
  - Giá trị phổ biến: 1, 3, 5, 7, 14, 30, 60, 90, 120, 180, 365
  - `7` days đủ cho debugging, tiết kiệm chi phí

#### 📅 EventBridge Scheduler Configuration

```hcl
manifest_bucket = "langfuse-ap-southeast-1-302010997939"
manifest_key    = "manifest.json"
```
- **manifest_bucket**: S3 bucket name chứa manifest.json
  - Duplicate với Lambda env var để Terraform biết dependencies
  - Dùng trong IAM policy và module configuration

- **manifest_key**: S3 object key của manifest file
  - Duplicate với Lambda env var cho consistency
  - Dùng trong module configuration

```hcl
schedule_expression = "rate(1 hour)"
```
- **schedule_expression**: EventBridge schedule expression
  - **Rate-based syntax**:
    - `rate(1 minute)` - Mỗi phút
    - `rate(5 minutes)` - Mỗi 5 phút
    - `rate(1 hour)` - Mỗi giờ
    - `rate(12 hours)` - Mỗi 12 giờ
    - `rate(1 day)` - Mỗi ngày
  
  - **Cron-based syntax**: `cron(Minutes Hours Day-of-month Month Day-of-week Year)`
    - `cron(0 12 * * ? *)` - 12:00 PM UTC mỗi ngày
    - `cron(0 0 * * ? *)` - 12:00 AM UTC mỗi ngày
    - `cron(0/30 * * * ? *)` - Mỗi 30 phút
    - `cron(0 9 ? * MON-FRI *)` - 9:00 AM UTC thứ 2-6
  
  - **Lưu ý**: 
    - EventBridge dùng UTC timezone
    - Rate expressions phù hợp cho regular intervals
    - Cron expressions linh hoạt hơn cho specific schedules

### 2. Build Lambda Package

Lambda function sử dụng native dependencies (psycopg2) nên cần build bằng Docker:

```bash
cd services/lambda-s3-processor

# Build with Docker (recommended)
./build_lambda.sh --docker

# Or specify build image
./build_lambda.sh --docker --image lambci
./build_lambda.sh --docker --image amazonlinux
```

**Build script options:**
- `--docker`: Use Docker for building (required for native dependencies)
- `--image lambci`: Use `lambci/lambda:build-python3.11` image (default)
- `--image amazonlinux`: Use `amazonlinux:2` image (alternative)

**Script output:**
- Creates `lambda/manifest-processor.zip` with all dependencies
- Compatible với AWS Lambda Python 3.11 runtime
- Includes: `index.py`, `psycopg2-binary`, `requests`, `boto3`

### 3. Deploy with Terraform

```bash
cd services/lambda-s3-processor

# Initialize Terraform
terraform init

# Review changes
terraform plan

# Deploy infrastructure
terraform apply

# Verify deployment
terraform output
```

## 📝 Lambda Function Code

Lambda function code trong `lambda/manifest-processor/index.py` thực hiện các công việc sau:

### 🔑 Core Functions

#### 1. `lambda_handler(event, context)`
- **Entry point** của Lambda function
- Đọc `MANIFEST_BUCKET` và `MANIFEST_KEY` từ environment variables
- Gọi `sync_om_metadata()` để xử lý manifest và đồng bộ lineage
- Return status code và message

#### 2. `get_secret(secret_name)`
- Lấy secret value từ AWS Secrets Manager
- Support JSON secrets (DB credentials) và plain text secrets (API tokens)
- Sử dụng `boto3.client('secretsmanager')`

#### 3. `get_db_conn()`
- Tạo PostgreSQL connection đến OpenMetadata database
- Sử dụng credentials từ `OM_DB_SECRET`
- Return `psycopg2` connection object

#### 4. `inti_db_resource()`
- Tạo bảng `om_lineage_state` nếu chưa tồn tại
- Schema:
  ```sql
  CREATE TABLE IF NOT EXISTS public.om_lineage_state (
    from_fqn VARCHAR(512),
    to_fqn VARCHAR(512),
    PRIMARY KEY (from_fqn, to_fqn)
  )
  ```
- Dùng để track lineage đã tạo, tránh duplicates

#### 5. `parsing_lineage_table_level(manifest_data)`
- Parse dbt manifest.json để extract table-level lineage
- Tìm dependencies giữa các models (tables)
- Return list các lineage edges: `[(from_fqn, to_fqn), ...]`
- Format FQN: `database.schema.table`

#### 6. `create_hub_spoke_lineage()`
- **Tự động tạo lineage** giữa hub và spoke databases
- **SQL Query**:
  ```sql
  SELECT hub.id, hub.name, spoke.id, hub.json->>'fullyQualifiedName', spoke.json->>'fullyQualifiedName'
  FROM table_entity hub
  INNER JOIN table_entity spoke ON hub.name = spoke.name
  WHERE hub.json->'database'->>'name' = 'hub_database_name'
    AND spoke.json->'database'->>'name' = 'spoke_database_name'
  ```
- Tạo lineage **spoke → hub** cho mỗi cặp tables matching
- Call OpenMetadata API: `PUT /v1/lineage`
- Lưu vào `om_lineage_state` với `ON CONFLICT DO NOTHING`
- Return số lượng lineage đã tạo

#### 7. `run_pipeline_sync_lineage(manifest_data)`
- **Main lineage sync logic** cho dbt manifest
- **Bước 1**: Parse manifest để lấy lineage mới
- **Bước 2**: Query `om_lineage_state` để lấy lineage hiện tại
- **Bước 3**: So sánh và tìm lineage cần delete (outdated)
- **Bước 4**: Delete outdated lineage:
  - Call API: `DELETE /v1/lineage/{from}/to/{to}`
  - **Chỉ delete từ state table sau khi API call thành công**
- **Bước 5**: Tạo lineage mới:
  - Call API: `PUT /v1/lineage`
  - Insert vào `om_lineage_state` on success
- **Error handling**: Continue processing nếu 1 lineage fail

#### 8. `sync_om_metadata(bucket_name, object_key)`
- **Orchestrator function** - điều phối toàn bộ workflow
- Download manifest.json từ S3
- Initialize database resources
- Run dbt manifest lineage sync
- Run hub-spoke cross-database lineage
- Close connections

### 📦 Dependencies

File `requirements.txt`:
```
requests>=2.31.0
psycopg2-binary>=2.9.9
boto3>=1.34.0
```

- **requests**: HTTP client cho OpenMetadata REST API calls
- **psycopg2-binary**: PostgreSQL adapter với native binaries
- **boto3**: AWS SDK cho S3 và Secrets Manager

### 🔄 Processing Flow

```
lambda_handler()
    │
    ├─► Đọc MANIFEST_BUCKET, MANIFEST_KEY từ env vars
    │
    └─► sync_om_metadata(bucket, key)
            │
            ├─► Download manifest.json từ S3
            │
            ├─► Get DB credentials từ Secrets Manager
            │
            ├─► Connect to OpenMetadata PostgreSQL
            │
            ├─► inti_db_resource() - Tạo om_lineage_state table
            │
            ├─► run_pipeline_sync_lineage(manifest_data)
            │       │
            │       ├─► parsing_lineage_table_level() - Parse manifest
            │       │
            │       ├─► Query om_lineage_state - Lấy lineage hiện tại
            │       │
            │       ├─► Compare và tìm outdated lineage
            │       │
            │       ├─► DELETE outdated lineage via API
            │       │   └─► Xóa từ om_lineage_state sau khi thành công
            │       │
            │       └─► PUT new lineage via API
            │           └─► INSERT vào om_lineage_state on success
            │
            ├─► create_hub_spoke_lineage()
            │       │
            │       ├─► Query OpenMetadata DB - Tìm matching tables
            │       │   (INNER JOIN hub.name = spoke.name)
            │       │
            │       ├─► PUT lineage (spoke → hub) via API
            │       │
            │       └─► INSERT vào om_lineage_state
            │
            └─► Close connections và return success
```

## 🔧 Configuration Options

### Lambda Functions

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| `function_name` | string | Lambda function name (sẽ được thêm prefix) | ✅ Yes | - |
| `description` | string | Function description | ✅ Yes | - |
| `handler` | string | Lambda handler (e.g., index.lambda_handler) | ✅ Yes | - |
| `runtime` | string | Runtime (python3.11, nodejs20.x, etc.) | ✅ Yes | - |
| `timeout` | number | Function timeout (seconds, max 900) | ✅ Yes | - |
| `memory_size` | number | Memory in MB (128-10240) | ✅ Yes | - |
| `filename` | string | Path to ZIP file | ❌ No | null |
| `s3_bucket` | string | S3 bucket for code deployment | ❌ No | null |
| `s3_key` | string | S3 key for code deployment | ❌ No | null |
| `environment.variables` | map(string) | Environment variables | ❌ No | {} |
| `logs_retention_days` | number | CloudWatch Logs retention (days) | ❌ No | 7 |

**Lưu ý**:
- Phải có **hoặc** `filename` (local ZIP) **hoặc** `s3_bucket` + `s3_key` (S3-based deployment)
- `timeout` max = 900 seconds (15 minutes)
- `memory_size` affects CPU allocation (more memory = more CPU)
- `logs_retention_days` values: 1, 3, 5, 7, 14, 30, 60, 90, 120, 180, 365, 400, 545, 731, 1827, 3653

### EventBridge Scheduler

| Parameter | Type | Description | Required | Default | Example |
|-----------|------|-------------|----------|---------|---------|
| `schedule_expression` | string | Schedule expression (rate or cron) | ✅ Yes | - | `rate(1 hour)` |
| `manifest_bucket` | string | S3 bucket name | ✅ Yes | - | `my-bucket` |
| `manifest_key` | string | S3 object key | ✅ Yes | - | `manifest.json` |

**Schedule Expression Syntax**:
- **Rate**: `rate(value unit)` where unit is `minute(s)`, `hour(s)`, `day(s)`
  - `rate(5 minutes)` - Every 5 minutes
  - `rate(1 hour)` - Every hour
  - `rate(2 hours)` - Every 2 hours
  - `rate(1 day)` - Every day

- **Cron**: `cron(Minutes Hours Day-of-month Month Day-of-week Year)`
  - `cron(0 12 * * ? *)` - At 12:00 PM UTC every day
  - `cron(15 10 ? * MON-FRI *)` - At 10:15 AM UTC Monday-Friday
  - `cron(0 0/4 * * ? *)` - Every 4 hours starting at midnight
  - `?` in Day-of-month or Day-of-week means "any"

### VPC Configuration

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `vpc_subnet_ids` | list(string) | Private subnet IDs | ✅ Yes |
| `vpc_security_group_ids` | list(string) | Security group IDs | ✅ Yes |

**Requirements**:
- Subnets **must** be private with NAT Gateway route
- Security groups need outbound rules:
  - Port 443 → 0.0.0.0/0 (AWS APIs)
  - Port 5432 → OpenMetadata DB (if applicable)
- Subnets need sufficient IP addresses

### IAM Configuration

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| `iam_role_name` | string | IAM role name (sẽ được thêm prefix) | ✅ Yes | - |
| `s3_bucket_arns` | list(string) | S3 bucket ARNs for access | ✅ Yes | - |
| `role_policy_arns` | list(string) | Additional managed policy ARNs | ❌ No | [] |

**Built-in Permissions**:
- S3: `s3:GetObject`, `s3:ListBucket` on specified buckets
- Secrets Manager: `secretsmanager:GetSecretValue` on all secrets
- CloudWatch Logs: Create log groups, streams, and put events
- VPC: Create/describe/delete ENIs

### Environment Variables (Lambda)

| Variable | Description | Required | Example |
|----------|-------------|----------|---------|
| `LOG_LEVEL` | Logging level | ❌ No | `INFO` |
| `MAIN_URL` | OpenMetadata API base URL | ✅ Yes | `https://om.example.com/api` |
| `OM_DB_SECRET` | Secrets Manager secret for DB credentials | ✅ Yes | `prod/om-db` |
| `OM_TOKEN_SECRET` | Secrets Manager secret for API token | ✅ Yes | `prod/om-token` |
| `MANIFEST_BUCKET` | S3 bucket name | ✅ Yes | `my-bucket` |
| `MANIFEST_KEY` | S3 object key | ✅ Yes | `manifest.json` |
| `HUB_DATABASE` | Hub database name in OpenMetadata | ✅ Yes | `hub_database` |
| `SPOKE_DATABASE` | Spoke database name in OpenMetadata | ✅ Yes | `spoke_database` |

## 📤 Outputs

| Output | Description |
|--------|-------------|
| `iam_role_arn` | IAM role ARN |
| `iam_role_name` | IAM role name |
| `lambda_function_arns` | Map of Lambda function ARNs |
| `lambda_function_names` | Map of Lambda function names |
| `lambda_function_invoke_arns` | Map of Lambda invoke ARNs |

## 🔐 IAM Permissions

Lambda IAM role được tạo tự động với các permissions sau:

### 📋 Built-in Policies

#### 1. S3 Access
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::bucket-name",
    "arn:aws:s3:::bucket-name/*"
  ]
}
```
- Read access to specified S3 buckets (from `s3_bucket_arns` variable)
- Both bucket-level and object-level permissions

#### 2. Secrets Manager Access
```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue"
  ],
  "Resource": "*"
}
```
- Get secret values from AWS Secrets Manager
- Used for DB credentials và API tokens

#### 3. CloudWatch Logs
```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Resource": "arn:aws:logs:*:*:*"
}
```
- Create and write to CloudWatch Logs
- Automatic log group creation

#### 4. VPC Access (AWS Managed)
Attached policy: `arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole`

Includes:
```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:CreateNetworkInterface",
    "ec2:DescribeNetworkInterfaces",
    "ec2:DeleteNetworkInterface",
    "ec2:AssignPrivateIpAddresses",
    "ec2:UnassignPrivateIpAddresses"
  ],
  "Resource": "*"
}
```
- Create/manage ENIs in VPC subnets
- Required for VPC-enabled Lambda

### 🔧 Trust Relationship

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
