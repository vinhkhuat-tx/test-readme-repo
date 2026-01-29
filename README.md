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

### ➕ Additional Policies

Có thể thêm AWS managed policies qua `role_policy_arns`:

```hcl
role_policy_arns = [
  "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess",
  "arn:aws:iam::aws:policy/AmazonRDSReadOnlyAccess"
]
```

## 🌐 VPC Configuration

Lambda functions chạy trong VPC để access private resources (databases, internal APIs).

### 📋 Requirements

#### 1. Private Subnets
- **Bắt buộc**: Subnets must be **private** (không có direct internet gateway route)
- **NAT Gateway**: Subnets cần route table có route đến NAT Gateway
  ```
  Destination: 0.0.0.0/0
  Target: nat-xxxxx
  ```
- **Lý do**: Lambda cần internet để call AWS APIs (S3, Secrets Manager, CloudWatch)

#### 2. Security Groups
**Outbound Rules Required**:
```
Type: HTTPS
Protocol: TCP
Port: 443
Destination: 0.0.0.0/0
Description: AWS APIs (S3, Secrets Manager)

Type: PostgreSQL
Protocol: TCP
Port: 5432
Destination: sg-xxxxx (OpenMetadata DB security group)
Description: OpenMetadata Database
```

**Inbound Rules**: Không cần (trừ khi Lambda expose service)

#### 3. IP Addresses
- Mỗi Lambda ENI cần 1 IP address
- Concurrent executions = số ENIs
- **Khuyến nghị**: Subnet size /24 hoặc lớn hơn
- **Tính toán**: 
  - /24 subnet = 256 IPs - 5 AWS reserved = 251 available
  - Nếu max concurrent = 100 → cần ~100 IPs

### 🔍 VPC Lambda Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      VPC                                │
│                                                         │
│  ┌──────────────────────┐    ┌──────────────────────┐  │
│  │  Private Subnet A    │    │  Private Subnet B    │  │
│  │  (AZ 1)              │    │  (AZ 2)              │  │
│  │                      │    │                      │  │
│  │  ┌────────────────┐  │    │  ┌────────────────┐  │  │
│  │  │ Lambda ENI     │  │    │  │ Lambda ENI     │  │  │
│  │  │ (10.0.1.x)     │  │    │  │ (10.0.2.x)     │  │  │
│  │  └────────┬───────┘  │    │  └────────┬───────┘  │  │
│  │           │          │    │           │          │  │
│  └───────────┼──────────┘    └───────────┼──────────┘  │
│              │                           │             │
│              └───────────┬───────────────┘             │
│                          │                             │
│              ┌───────────▼──────────┐                  │
│              │  Security Group      │                  │
│              │  - Outbound: 443     │                  │
│              │  - Outbound: 5432    │                  │
│              └───────────┬──────────┘                  │
│                          │                             │
│              ┌───────────▼──────────┐                  │
│              │  NAT Gateway         │                  │
│              │  (in Public Subnet)  │                  │
│              └───────────┬──────────┘                  │
└──────────────────────────┼──────────────────────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Internet Gateway│
                  └────────┬────────┘
                           │
                           ▼
        ┌──────────────────────────────────┐
        │  AWS Services                    │
        │  - S3                            │
        │  - Secrets Manager               │
        │  - CloudWatch Logs               │
        └──────────────────────────────────┘
```

### ⚠️ Important Notes

1. **Cold Start**: VPC Lambda có cold start time ~10-15s (tạo ENI)
2. **ENI Cleanup**: AWS tự động delete ENIs sau khi Lambda xóa
3. **Hyperplane ENI**: AWS dùng Hyperplane để share ENIs giữa functions (faster cold start)
4. **IP Exhaustion**: Monitor subnet IP usage, scale subnets nếu cần
5. **NAT Gateway Cost**: NAT Gateway charge per GB data processed (~$0.045/GB)

### 🔧 Troubleshooting VPC Issues

#### Lambda timeout connecting to database
- ✅ Check security group outbound rules allow port 5432
- ✅ Check database security group inbound rules allow Lambda SG
- ✅ Verify subnets have route to database (same VPC or peering)

#### Lambda cannot access S3/Secrets Manager
- ✅ Check NAT Gateway exists and is active
- ✅ Check route table has 0.0.0.0/0 → NAT Gateway
- ✅ Check security group allows outbound port 443

#### Insufficient IP addresses
- ✅ Use larger subnet (/23 instead of /24)
- ✅ Reduce reserved concurrency on Lambda
- ✅ Use multiple subnets across AZs

## 📊 Monitoring & Observability

### CloudWatch Logs

Lambda logs được tự động stream đến CloudWatch Logs:

```
Log Group: /aws/lambda/{env}-{last4digits}-{region_short}-{function-name}
Example: /aws/lambda/dev-7939-apse1-manifest-processor
```

**Log Retention**: Configurable trong `terraform.tfvars` (default: 7 days)

**Log Format**:
```
[INFO] 2024-01-29 10:15:23 - Starting manifest processing
[INFO] 2024-01-29 10:15:24 - Downloaded manifest from s3://bucket/manifest.json
[INFO] 2024-01-29 10:15:25 - Connected to OpenMetadata database
[INFO] 2024-01-29 10:15:26 - Found 45 lineage edges in manifest
[INFO] 2024-01-29 10:15:27 - Deleting 3 outdated lineage edges
[INFO] 2024-01-29 10:15:30 - Creating 5 new lineage edges
[INFO] 2024-01-29 10:15:35 - Created 12 hub-spoke lineage edges
[INFO] 2024-01-29 10:15:36 - Processing completed successfully
```

### CloudWatch Metrics

**Available Metrics** (automatically collected):

| Metric | Description | Unit |
|--------|-------------|------|
| `Invocations` | Number of times function invoked | Count |
| `Duration` | Execution time | Milliseconds |
| `Errors` | Number of invocations with errors | Count |
| `Throttles` | Number of throttled invocations | Count |
| `ConcurrentExecutions` | Number of instances processing events | Count |
| `IteratorAge` | For stream-based invocations | Milliseconds |

**Custom Metrics**: Có thể publish custom metrics bằng boto3:
```python
import boto3

cloudwatch = boto3.client('cloudwatch')
cloudwatch.put_metric_data(
    Namespace='Lambda/ManifestProcessor',
    MetricData=[{
        'MetricName': 'LineageCreated',
        'Value': 42,
        'Unit': 'Count'
    }]
)
```

### X-Ray Tracing (Optional)

Enable X-Ray để trace requests qua multiple services:

```hcl
lambda_functions = {
  "manifest-processor" = {
    # ... other config
    tracing_config = {
      mode = "Active"  # or "PassThrough"
    }
  }
}
```

Add IAM policy:
```hcl
role_policy_arns = [
  "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
]
```

### Alarms & Alerting

**Recommended CloudWatch Alarms**:

#### 1. Error Rate Alarm
```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "lambda-manifest-processor-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 5
  alarm_description   = "Lambda function has too many errors"
  
  dimensions = {
    FunctionName = module.lambda_s3_processor.lambda_function_names["manifest-processor"]
  }
}
```

#### 2. Duration Alarm
```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_duration" {
  alarm_name          = "lambda-manifest-processor-duration"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Average"
  threshold           = 240000  # 4 minutes (timeout = 5 minutes)
  alarm_description   = "Lambda function taking too long"
  
  dimensions = {
    FunctionName = module.lambda_s3_processor.lambda_function_names["manifest-processor"]
  }
}
```

#### 3. Throttle Alarm
```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_throttles" {
  alarm_name          = "lambda-manifest-processor-throttles"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Throttles"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Lambda function is being throttled"
  
  dimensions = {
    FunctionName = module.lambda_s3_processor.lambda_function_names["manifest-processor"]
  }
}
```

### Dashboards

**Sample CloudWatch Dashboard**:
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Invocations", {"stat": "Sum"}],
          [".", "Errors", {"stat": "Sum"}],
          [".", "Duration", {"stat": "Average"}]
        ],
        "period": 300,
        "stat": "Average",
        "region": "ap-southeast-1",
        "title": "Lambda Manifest Processor Metrics"
      }
    }
  ]
}
```

## 🎯 Use Cases & Examples

### 1. Hourly Manifest Sync (Default)

**Scenario**: Đồng bộ dbt manifest lineage mỗi giờ

```hcl
schedule_expression = "rate(1 hour)"

lambda_functions = {
  "manifest-processor" = {
    timeout     = 300
    memory_size = 512
    environment = {
      variables = {
        MANIFEST_BUCKET = "my-dbt-bucket"
        MANIFEST_KEY    = "target/manifest.json"
      }
    }
  }
}
```

### 2. Daily Night Sync

**Scenario**: Chạy 1 lần mỗi đêm lúc 2:00 AM UTC

```hcl
schedule_expression = "cron(0 2 * * ? *)"

lambda_functions = {
  "manifest-processor" = {
    timeout     = 600  # 10 minutes for large manifests
    memory_size = 1024
    environment = {
      variables = {
        MANIFEST_BUCKET = "production-dbt-bucket"
        MANIFEST_KEY    = "manifests/prod_manifest.json"
      }
    }
  }
}
```

### 3. Business Hours Only

**Scenario**: Chạy mỗi 30 phút trong giờ làm việc (9 AM - 6 PM UTC, Mon-Fri)

```hcl
# Cần multiple EventBridge rules hoặc dùng Step Functions
# Đơn giản hơn: rate(30 minutes) và check time trong Lambda code

schedule_expression = "rate(30 minutes)"

lambda_functions = {
  "manifest-processor" = {
    environment = {
      variables = {
        BUSINESS_HOURS_ONLY = "true"
        BUSINESS_START_HOUR = "9"
        BUSINESS_END_HOUR   = "18"
      }
    }
  }
}
```

### 4. Multiple Environments

**Scenario**: Separate Lambda cho từng environment

```hcl
lambda_functions = {
  "manifest-processor-dev" = {
    function_name = "manifest-processor-dev"
    environment = {
      variables = {
        MANIFEST_BUCKET = "dev-dbt-bucket"
        HUB_DATABASE    = "dev_hub"
        SPOKE_DATABASE  = "dev_spoke"
      }
    }
    logs_retention_days = 3
  }
  
  "manifest-processor-prod" = {
    function_name = "manifest-processor-prod"
    environment = {
      variables = {
        MANIFEST_BUCKET = "prod-dbt-bucket"
        HUB_DATABASE    = "prod_hub"
        SPOKE_DATABASE  = "prod_spoke"
      }
    }
    logs_retention_days = 30
  }
}

# Separate schedules
# Dev: Every 2 hours
# Prod: Every hour
```

### 5. Large Manifest Processing

**Scenario**: Manifest lớn (>10MB), nhiều tables (>1000)

```hcl
lambda_functions = {
  "manifest-processor" = {
    timeout     = 900  # Max 15 minutes
    memory_size = 2048 # 2GB RAM for processing
    
    environment = {
      variables = {
        LOG_LEVEL       = "WARNING"  # Reduce logging overhead
        BATCH_SIZE      = "100"      # Process lineage in batches
      }
    }
  }
}
```

### 6. Hub-Spoke Only (No Manifest)

**Scenario**: Chỉ tạo hub-spoke lineage, không process manifest

```python
# Modify index.py
def sync_om_metadata(bucket_name, object_key):
    # Skip manifest processing
    # manifest_data = download_manifest(bucket_name, object_key)
    # run_pipeline_sync_lineage(manifest_data)
    
    # Only run hub-spoke
    create_hub_spoke_lineage()
```

```hcl
schedule_expression = "rate(12 hours)"  # Less frequent

lambda_functions = {
  "hub-spoke-lineage" = {
    function_name = "hub-spoke-lineage"
    timeout       = 60  # Faster without manifest
    memory_size   = 256
  }
}
```

### 7. Multi-Region Deployment

**Scenario**: Deploy trong multiple AWS regions

```hcl
# us-east-1
provider "aws" {
  region = "us-east-1"
  alias  = "us_east_1"
}

module "lambda_processor_us_east_1" {
  source = "../../modules/layer/lambda-s3-processor"
  providers = {
    aws = aws.us_east_1
  }
  # ... config
}

# ap-southeast-1
provider "aws" {
  region = "ap-southeast-1"
  alias  = "ap_southeast_1"
}

module "lambda_processor_apse1" {
  source = "../../modules/layer/lambda-s3-processor"
  providers = {
    aws = aws.ap_southeast_1
  }
  # ... config
}
```

## 🐛 Troubleshooting

### Lambda Execution Issues

#### ❌ Lambda timeout (Task timed out after 300.00 seconds)

**Nguyên nhân**:
- Manifest file quá lớn
- Quá nhiều lineage edges cần process
- Database connection slow
- Network latency trong VPC

**Giải pháp**:
```hcl
lambda_functions = {
  "manifest-processor" = {
    timeout     = 600  # Tăng lên 10 minutes
    memory_size = 1024 # Tăng RAM để xử lý nhanh hơn
  }
}
```

#### ❌ Out of memory error

**Nguyên nhân**:
- Memory_size quá thấp cho manifest size
- Memory leak trong code

**Giải pháp**:
```hcl
lambda_functions = {
  "manifest-processor" = {
    memory_size = 1024  # Tăng từ 512 → 1024
  }
}
```

Hoặc optimize code:
```python
# Process lineage in batches
BATCH_SIZE = 100
for i in range(0, len(lineages), BATCH_SIZE):
    batch = lineages[i:i+BATCH_SIZE]
    process_batch(batch)
```

#### ❌ Cannot import module 'index'

**Nguyên nhân**:
- Build Lambda package không đúng
- Missing dependencies

**Giải pháp**:
```bash
# Rebuild with Docker
cd services/lambda-s3-processor
./build_lambda.sh --docker

# Verify ZIP contents
unzip -l lambda/manifest-processor.zip
```

#### ❌ No module named 'psycopg2._psycopg'

**Nguyên nhân**:
- psycopg2 compiled trên host machine không tương thích với Lambda

**Giải pháp**:
```bash
# MUST build with Docker
./build_lambda.sh --docker
```

### VPC Connectivity Issues

#### ❌ Lambda cannot connect to database (timeout)

**Check list**:
1. **Security group outbound rules**:
   ```bash
   aws ec2 describe-security-groups --group-ids sg-xxxxx
   # Verify: Port 5432 → database SG
   ```

2. **Database security group inbound rules**:
   ```bash
   # Verify: Port 5432 FROM lambda SG
   ```

3. **Network connectivity**:
   ```bash
   # Test from EC2 in same subnet
   telnet db-host 5432
   ```

#### ❌ Lambda cannot access S3 (403 Forbidden)

**Check list**:
1. **IAM permissions**:
   ```bash
   aws lambda get-policy --function-name function-name
   ```

2. **S3 bucket policy**:
   ```bash
   aws s3api get-bucket-policy --bucket bucket-name
   ```

3. **NAT Gateway routing**:
   ```bash
   aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=subnet-xxxxx"
   # Verify: 0.0.0.0/0 → nat-xxxxx
   ```

#### ❌ Insufficient IP addresses in subnet

**Error**: `ENI could not be created because subnet has insufficient free addresses`

**Giải pháp**:
1. Use larger subnet (/23 instead of /24)
2. Add more subnets:
   ```hcl
   vpc_subnet_ids = [
     "subnet-xxxxx",
     "subnet-yyyyy",
     "subnet-zzzzz"  # Add more
   ]
   ```

3. Reduce reserved concurrency:
   ```hcl
   reserved_concurrent_executions = 10
   ```

### AWS Secrets Manager Issues

#### ❌ Secret not found

**Nguyên nhân**:
- Secret name trong env var không đúng
- Secret không tồn tại trong region

**Giải pháp**:
```bash
# List secrets
aws secretsmanager list-secrets --region ap-southeast-1

# Verify secret name matches env var
OM_DB_SECRET = "prod/om-db"  # Must match secret name
```

#### ❌ Access denied to secret

**Giải pháp**:
```bash
# Check IAM policy includes secretsmanager:GetSecretValue
aws iam get-role-policy --role-name role-name --policy-name policy-name
```

### EventBridge Scheduler Issues

#### ❌ Lambda not being triggered

**Check list**:
1. **EventBridge rule status**:
   ```bash
   aws events describe-rule --name rule-name
   # State should be "ENABLED"
   ```

2. **EventBridge target**:
   ```bash
   aws events list-targets-by-rule --rule rule-name
   # Verify Lambda ARN is correct
   ```

3. **Lambda permission**:
   ```bash
   aws lambda get-policy --function-name function-name
   # Verify events.amazonaws.com has invoke permission
   ```

4. **Check CloudWatch Logs**:
   ```bash
   aws logs tail /aws/lambda/function-name --follow
   ```

#### ❌ Schedule not running at expected time

**Nguyên nhân**:
- EventBridge dùng UTC timezone
- Cron expression không đúng

**Giải pháp**:
```hcl
# Convert local time to UTC
# Singapore (UTC+8): 9:00 AM → 1:00 AM UTC
schedule_expression = "cron(0 1 * * ? *)"

# Test schedule expression
# https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html
```

### OpenMetadata API Issues

#### ❌ 401 Unauthorized

**Nguyên nhân**:
- API token expired hoặc invalid
- Token không được include trong request header

**Giải pháp**:
```bash
# Test API token manually
curl -H "Authorization: Bearer $TOKEN" https://om-host/api/v1/users

# Verify token in Secrets Manager
aws secretsmanager get-secret-value --secret-id prod/om-token
```

#### ❌ 404 Not Found (lineage endpoints)

**Nguyên nhân**:
- FQN (Fully Qualified Name) không đúng format
- Table entity không tồn tại trong OpenMetadata

**Giải pháp**:
```python
# Verify FQN format
# Correct: "database.schema.table"
# Wrong: "database:schema:table"

# Check table exists
SELECT json->>'fullyQualifiedName' 
FROM table_entity 
WHERE name = 'table_name';
```

### Database Connection Issues

#### ❌ Connection refused

**Check list**:
1. **Database host/port correct**:
   ```python
   # Verify secret value
   {
     "host": "correct-host.rds.amazonaws.com",
     "port": 5432
   }
   ```

2. **Security group allows inbound**:
   ```bash
   aws ec2 describe-security-groups --group-ids sg-db-xxxxx
   ```

3. **Database is running**:
   ```bash
   aws rds describe-db-instances --db-instance-identifier db-id
   ```

#### ❌ Authentication failed

**Nguyên nhân**:
- Username/password không đúng trong Secrets Manager
- Database user không có permissions

**Giải pháp**:
```sql
-- Grant permissions to OM user
GRANT SELECT, INSERT, DELETE ON table_entity TO om_user;
GRANT ALL ON om_lineage_state TO om_user;
```

### Terraform Issues

#### ❌ No changes detected after rebuilding Lambda

**Nguyên nhân**:
- `source_code_hash` không thay đổi

**Giải pháp**: Already fixed in `main.tf`:
```hcl
source_code_hash = config.filename != null ? filebase64sha256(config.filename) : config.source_code_hash
```

#### ❌ EventBridge rule name too long

**Nguyên nhân**:
- Rule name exceeds 64 characters

**Giải pháp**: Already fixed with truncation:
```hcl
name = length(format(...)) > 64 ? substr(format(...), 0, 64) : format(...)
```

### Performance Optimization

#### 🐌 Lambda execution taking too long

**Optimization strategies**:

1. **Increase memory** (also increases CPU):
   ```hcl
   memory_size = 1024  # or 2048
   ```

2. **Process lineage in parallel**:
   ```python
   from concurrent.futures import ThreadPoolExecutor
   
   with ThreadPoolExecutor(max_workers=10) as executor:
       futures = [executor.submit(create_lineage, edge) for edge in edges]
   ```

3. **Use database connection pooling**:
   ```python
   from psycopg2 import pool
   
   connection_pool = pool.SimpleConnectionPool(1, 10, **db_config)
   ```

4. **Cache API tokens**:
   ```python
   _token_cache = None
   
   def get_token():
       global _token_cache
       if _token_cache is None:
           _token_cache = get_secret(os.environ['OM_TOKEN_SECRET'])
       return _token_cache
   ```

### Debugging Tips

#### 📝 Enable debug logging

```hcl
environment = {
  variables = {
    LOG_LEVEL = "DEBUG"
  }
}
```

#### 🔍 Test Lambda locally

```bash
# Use AWS SAM CLI
sam local invoke -e event.json

# Or use Python directly
python -c "from index import lambda_handler; lambda_handler({'bucket': '...', 'key': '...'}, None)"
```

#### 📊 Monitor CloudWatch metrics

```bash
# View recent invocations
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Invocations \
  --dimensions Name=FunctionName,Value=function-name \
  --start-time 2024-01-29T00:00:00Z \
  --end-time 2024-01-29T23:59:59Z \
  --period 3600 \
  --statistics Sum
```

## 📚 References

- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/)
- [EventBridge Scheduled Rules](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html)
- [Lambda in VPC](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [OpenMetadata API Documentation](https://docs.open-metadata.org/swagger.html)
- [psycopg2 Documentation](https://www.psycopg.org/docs/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

## 💰 Cost Estimation

### Lambda Costs

**Pricing Model** (ap-southeast-1 region):
- **Requests**: $0.20 per 1M requests
- **Duration**: $0.0000166667 per GB-second

**Example**: 512MB, 60s execution, triggered hourly (720 times/month):
```
Requests: 720 × $0.20/1M = $0.000144
Duration: 720 × (512/1024) × 60 × $0.0000166667 = $0.36
Total: ~$0.36/month
```

**Example**: 1GB, 300s execution, triggered hourly:
```
Requests: $0.000144
Duration: 720 × 1 × 300 × $0.0000166667 = $3.60
Total: ~$3.60/month
```

### VPC Costs

**NAT Gateway** (if used):
- Hourly charge: $0.045/hour = $32.85/month
- Data processing: $0.045/GB
- **Note**: Most expensive component!

**VPC Endpoints** (alternative to NAT for AWS services):
- Interface endpoint: $0.01/hour = $7.30/month per endpoint
- S3 Gateway endpoint: **FREE**
- Data transfer: $0.01/GB

**Cost optimization**:
```hcl
# Use S3 Gateway Endpoint (free) instead of NAT for S3
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = var.vpc_id
  service_name = "com.amazonaws.ap-southeast-1.s3"
  route_table_ids = [var.route_table_id]
}
```

### CloudWatch Logs

**Pricing**:
- Ingestion: $0.50 per GB
- Storage: $0.03 per GB/month
- **Free tier**: 5GB ingestion, 5GB storage per month

**Example**: 1MB logs per execution, 720 executions/month:
```
Data: 720 × 1MB = 720MB = 0.72GB
Ingestion: $0.50 × 0.72 = $0.36
Storage (7 days): ~$0.01
Total: ~$0.37/month
```

### Secrets Manager

**Pricing**:
- Secret storage: $0.40 per secret/month
- API calls: $0.05 per 10,000 requests

**Example**: 2 secrets, 720 Lambda invocations:
```
Storage: 2 × $0.40 = $0.80
API calls: 1440 × $0.05/10,000 = $0.007
Total: ~$0.81/month
```

### EventBridge

**Pricing**:
- Events: $1.00 per million events
- **Free tier**: 14 million events/month

**Example**: Hourly schedule (720 events/month):
```
Cost: 720 × $1.00/1M = $0.00072 (essentially free)
```

### Total Monthly Cost Estimate

**Small deployment** (512MB, 60s, hourly):
```
Lambda:          $0.36
NAT Gateway:     $32.85 (or VPC Endpoint: $7.30)
CloudWatch Logs: $0.37
Secrets Manager: $0.81
EventBridge:     $0.00
------------------------
Total:           $34.39 (or $8.84 with VPC endpoint)
```

**Medium deployment** (1GB, 300s, hourly):
```
Lambda:          $3.60
NAT Gateway:     $32.85 + data transfer
CloudWatch Logs: $1.50
Secrets Manager: $0.81
EventBridge:     $0.00
------------------------
Total:           ~$38.76+
```

**Cost optimization tips**:
1. ✅ Use S3 Gateway VPC Endpoint (free) instead of NAT
2. ✅ Reduce logs retention (3 days instead of 7)
3. ✅ Lower schedule frequency if possible
4. ✅ Optimize Lambda memory/timeout for actual needs
5. ✅ Use Reserved Concurrency to prevent runaway costs

## 🎓 Best Practices

### 1. Security

✅ **Use Secrets Manager** for sensitive data (never hardcode)
```hcl
environment = {
  variables = {
    DB_PASSWORD = "hardcoded"  # ❌ NEVER DO THIS
    DB_SECRET   = "prod/db"    # ✅ Use Secrets Manager
  }
}
```

✅ **Principle of Least Privilege** - IAM permissions
```hcl
# ❌ Too broad
s3_bucket_arns = ["arn:aws:s3:::*"]

# ✅ Specific buckets only
s3_bucket_arns = ["arn:aws:s3:::my-specific-bucket"]
```

✅ **VPC Security Groups** - Restrict outbound traffic
```
# ❌ Allow all
Outbound: 0.0.0.0/0 on all ports

# ✅ Specific ports only
Outbound: 0.0.0.0/0 on port 443 (HTTPS)
Outbound: 10.0.0.0/8 on port 5432 (PostgreSQL)
```

### 2. Reliability

✅ **Error Handling** - Graceful degradation
```python
try:
    create_lineage(edge)
except Exception as e:
    logger.error(f"Failed to create lineage: {e}")
    # Continue processing other lineages
    continue
```

✅ **Idempotency** - Safe to retry
```sql
-- Use ON CONFLICT to prevent duplicates
INSERT INTO om_lineage_state (from_fqn, to_fqn)
VALUES (%s, %s)
ON CONFLICT (from_fqn, to_fqn) DO NOTHING;
```

✅ **Dead Letter Queue** - Capture failed invocations
```hcl
lambda_functions = {
  "manifest-processor" = {
    dead_letter_config = {
      target_arn = aws_sqs_queue.dlq.arn
    }
  }
}
```

### 3. Performance

✅ **Right-size Lambda** - Match memory to workload
```hcl
# Small manifest (<1MB)
memory_size = 256
timeout     = 60

# Large manifest (>10MB)
memory_size = 1024
timeout     = 300
```

✅ **Connection Reuse** - Keep connections warm
```python
# ❌ Create new connection every time
def process():
    conn = psycopg2.connect(...)
    # process
    conn.close()

# ✅ Reuse connection across invocations
_db_connection = None

def get_connection():
    global _db_connection
    if _db_connection is None or _db_connection.closed:
        _db_connection = psycopg2.connect(...)
    return _db_connection
```

✅ **Batch Processing** - Process in chunks
```python
BATCH_SIZE = 100
for i in range(0, len(lineages), BATCH_SIZE):
    batch = lineages[i:i+BATCH_SIZE]
    process_batch(batch)
```

### 4. Observability

✅ **Structured Logging** - Machine-parseable logs
```python
import json
import logging

logger = logging.getLogger()

# ✅ JSON structured logs
logger.info(json.dumps({
    "event": "lineage_created",
    "from_fqn": from_fqn,
    "to_fqn": to_fqn,
    "duration_ms": duration
}))
```

✅ **Custom Metrics** - Track business metrics
```python
cloudwatch.put_metric_data(
    Namespace='ManifestProcessor',
    MetricData=[
        {
            'MetricName': 'LineageCreated',
            'Value': created_count,
            'Unit': 'Count'
        },
        {
            'MetricName': 'LineageDeleted',
            'Value': deleted_count,
            'Unit': 'Count'
        }
    ]
)
```

✅ **Alerting** - Be notified of issues
```hcl
resource "aws_cloudwatch_metric_alarm" "errors" {
  alarm_name          = "lambda-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  threshold           = 5
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

### 5. Cost Optimization

✅ **Schedule Frequency** - Run only when needed
```hcl
# ❌ Too frequent for daily updates
schedule_expression = "rate(5 minutes)"

# ✅ Match manifest update frequency
schedule_expression = "rate(1 hour)"
```

✅ **VPC Endpoints** - Reduce NAT costs
```hcl
# Use S3 Gateway Endpoint (free)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = var.vpc_id
  service_name = "com.amazonaws.${var.region}.s3"
}
```

✅ **Logs Retention** - Don't keep logs forever
```hcl
logs_retention_days = 7  # vs 365 days
# Saves: $0.03/GB/month × 11 months
```

### 6. Development Workflow

✅ **Local Testing** - Test before deploying
```bash
# Test Lambda handler locally
python -c "from index import lambda_handler; lambda_handler({'test': True}, None)"

# Use SAM for integration testing
sam local invoke -e event.json
```

✅ **CI/CD Pipeline** - Automated deployments
```yaml
# .github/workflows/deploy.yml
- name: Build Lambda
  run: ./build_lambda.sh --docker

- name: Terraform Apply
  run: |
    terraform init
    terraform apply -auto-approve
```

✅ **Version Control** - Track Lambda code changes
```bash
git tag -a v1.0.0 -m "Initial release"
git push --tags

# Reference in Terraform
source_code_hash = filebase64sha256("lambda-v1.0.0.zip")
```

## 📚 References

- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/)
- [EventBridge Scheduled Rules](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html)
- [Lambda in VPC](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [OpenMetadata API Documentation](https://docs.open-metadata.org/swagger.html)
- [psycopg2 Documentation](https://www.psycopg.org/docs/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
