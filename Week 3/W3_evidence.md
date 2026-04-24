# Week 3 Final Evidence

## 1) Cover

- **Group number**: 7

- **Group leader**: Phạm Hữu Tiến Thành

- **Members**:
  - Ngô Nguyễn Trường An
  - Nguyễn Văn Huy Hoàng
  - Nguyễn Phú Tài
  - Lê Thanh Sang
  - Phan Hoàng Nhật
  - Cái Xuân Hoà

- **Chosen database path** (engine + paradigm): **RDS MySQL / relational**
- **Back-link to Week 2 evidence**: https://github.com/thx2an/AWS-Project-each-week-GR4/blob/690a8c3491b27945b8d790e6fbfde65644cf3e3b/Week%202/W2_evidence.md

---

## W2 Recap: Kế thừa và Giải quyết Feedback từ W2

Tiếp nối kiến trúc từ Tuần 2, hệ thống vẫn duy trì nghiêm ngặt các tiêu chuẩn bảo mật nền tảng như Block Public Access cho S3, mã hóa mặc định (Default Encryption), Versioning và thiết lập IAM Baseline (MFA, Named Users, dọn dẹp các rule chứa Wildcards). 

Đặc biệt, nhóm đã ghi nhận và giải quyết triệt để các Trainer Feedback của W2 vào kiến trúc W3:

| # | Feedback từ Trainer | Trạng thái | Cách khắc phục |
|---|---|---|---|
| 1 | ECS task đang ở public subnet | ✅ **Đã sửa** | Chuyển toàn bộ ECS Fargate tasks vào Private Subnet. Chỉ ALB nằm ở Public Subnet nhận traffic từ internet. |
| 2 | ALB thiếu HTTPS/443 + ACM cert | ✅ **Đã sửa** | Bật HTTPS listener trên ALB, gắn ACM certificate cho domain. |
| 3 | ECR dùng public endpoint | ⚠️ **Chưa làm** | ECS pull image qua NAT Gateway (đi internet). VPC Interface Endpoint cho ECR chưa được tạo. |
| 4 | Secrets Manager injection chưa bảo mật | ✅ **Đã sửa** | Dùng `valueFrom` ARN trong ECS Task Definition — credentials không bao giờ xuất hiện plaintext. |
| 5 | DB Encryption (W1 feedback) | ✅ **Đã sửa** | RDS Encryption at Rest: **Enabled** (KMS aws/rds). Data lưu trên disk, snapshot và automated backup đều được mã hóa hoàn toàn. |


---

## 2) Data Access Pattern Log (Parts A, B, C)

### Part A - Access Patterns

1. **User Authentication** — Lookup user theo `email` để verify credentials và issue JWT token.
   - Frequency: ~20 calls/phút lúc peak.

2. **Product Catalog** — Fetch danh sách products JOIN với `categories` để lấy category name, lọc theo `shop_id` hoặc `slug`.
   - Frequency: ~100 calls/phút lúc peak.

3. **Order Processing** — Tạo order + decrement `products.stock` trong 1 ACID transaction đảm bảo inventory không bao giờ âm.
   - Frequency: ~10 calls/phút lúc peak.

### Part B - Engine + Paradigm + Reasoning

**Paradigm chọn:** Relational  
**Engine:** RDS MySQL Community 8.x (Managed)

**Tại sao Relational fit:**

- App có foreign key relationships: `users` → `orders` → `order_items` → `products` → `categories`. JOIN là bắt buộc, không thể tránh.
- ACID transactions cần thiết cho payment (VNPAY) và order processing: nếu decrement stock thành công nhưng insert order fail → phải rollback toàn bộ.
- Complex queries: lọc sản phẩm theo category, price range, sắp xếp theo rating — đây là use case điển hình của relational engine.

**Index đã tạo:**

- `users.email` — INDEX để lookup O(log N) thay vì full scan.
- `products.slug` — UNIQUE INDEX cho SEO-friendly URL lookup.
- `order_items.order_id` — INDEX hỗ trợ JOIN giữa `orders` và `order_items`.

**HA/Backup Plan:**

- **Multi-AZ: Yes** — RDS instance có standby ở AZ khác, tự động failover ~60 giây khi primary fail.
- Automated backup: 7 ngày retention, daily snapshot lúc 03:00 UTC.
- Point-in-time restore available.

**Tại sao chọn RDS MySQL Community:**

- Cost-effective cho giai đoạn học: `db.t3.micro` nằm trong AWS Free Tier.
- MySQL 8.x compatible hoàn toàn với TypeORM driver trong NestJS — không cần sửa code.
- Managed service: AWS lo OS patching, storage management, automated backup.


<!-- ### Part C - Data Model / Index Strategy



--- -->

## 3) Deployment Evidence

> Mỗi acceptance criterion = 1 entry với screenshot/CLI output + rationale.

### Criterion 1 - Subnet Group (Private Database Tier)

**Cấu hình:** RDS instance `minie-final-cluste` được đặt trong **private subnet** — không có route ra internet, Publicly accessible: **No**.

- Subnet Group Name: `minie-db-subnet-group`
- Subnets được chọn:
  - `DB-Subnet-1` — `10.0.2.0/24` — AZ `us-west-2a`
  - `DB-Subnet-2` — `10.0.3.0/24` — AZ `us-west-2b`
- Không có Internet Gateway route trong route table của 2 subnet này.

> **Config note:** "We placed the RDS instance in a private subnet with no IGW route and Publicly Accessible = No. Only the ECS application tier can reach port 3306 via the db-sg security group rule."

![RDS Subnet Configuration](docx/Database%20Layer/Screenshot/image.png)

#### Criterion 2 - Security Group (sg-rds)

**Cấu hình Inbound Rule:**

| Type         | Protocol | Port | Source                               |
| ------------ | -------- | ---- | ------------------------------------ |
| MySQL/Aurora | TCP      | 3306 | `sg-ecs` (Security Group ID của ECS) |

> **Config note:** "Inbound rule references the ECS Security Group ID directly — not a CIDR block. This means only tasks running in the ECS service can connect to Aurora. Even if someone gains access to the VPC, they cannot reach the DB without being in sg-ecs."

![RDS Security Group Inbound Rules](docx/Database%20Layer/Screenshot/image1.png)

---

### Criterion 3 - RDS MySQL Instance Creation

**Thông số cấu hình:**

| Parameter          | Giá trị                                    | Lý do                                                |
| ------------------ | ------------------------------------------ | ---------------------------------------------------- |
| Engine             | RDS MySQL Community 8.x                    | Compatible TypeORM/MySQL driver, AWS managed         |
| Template           | Free Tier                                  | Phù hợp môi trường học, cost thấp                    |
| DB Instance ID     | `minie-final-cluste`                       |                                                      |
| DB Instance Class  | `db.t3.micro`                              | Free Tier eligible, đủ cho workload học              |
| Multi-AZ           | **Enabled** (Yes)                          | Standby instance ở AZ khác, tự failover ~60s khi primary fail |
| DB Name            | `minie_db`                                 |                                                      |
| Encryption at Rest | **Enabled**                                | Bảo mật data lưu trên disk                           |
| Backup Retention   | 7 ngày                                     | Point-in-time restore, kết hợp với Multi-AZ đảm bảo HA    |
| Public Access      | **No** (Internet access gateway: Disabled) | Chỉ reach được từ trong VPC                          |
| VPC                | Mini-F-Final-vpc                           |                                                      |
| Subnet Group       | `minie-db-subnet-group`                    | Private subnets                                      |
| Security Group     | `db-sg` (sg-07e4bc33a8955b1944)            | Chỉ cho phép từ ECS security group                   |

> **Config note:** "We chose db.t3.micro (Free Tier) because our current workload does not require high memory. Multi-AZ is **enabled** — standby instance in another AZ provides automatic failover in ~60 seconds without manual intervention. 7-day automated backup provides point-in-time restore for data recovery scenarios."

![RDS Instance Configuration 1](docx/Database%20Layer/Screenshot/image2.png)
![RDS Instance Configuration 2](docx/Database%20Layer/Screenshot/image3.png)
![RDS Instance Configuration 3](docx/Database%20Layer/Screenshot/image4.png)
![RDS Instance Configuration 4](docx/Database%20Layer/Screenshot/image5.png)

---

### Criterion 4 - ECS Backend to RDS Connection via Secrets Manager

Credentials được lưu trong **AWS Secrets Manager** (không hardcode trong code):

| Secret Key | Giá trị (ẩn)                                                  |
| ---------- | ------------------------------------------------------------- |
| `DB_HOST`  | `minie-final-cluste.ci46ao2sg9te.us-west-2.rds.amazonaws.com` |
| `DB_PORT`  | `3306`                                                        |
| `DB_USER`  | `admin`                                                       |
| `DB_PASS`  | `********`                                                    |
| `DB_NAME`  | `minie_db`                                                    |

ECS Task Definition inject secrets từ Secrets Manager vào container environment variable thông qua `valueFrom` ARN — không có plain-text credential trong code hay console.

> **Config note:** "We use AWS Secrets Manager valueFrom injection so that credentials are never stored in the Docker image or ECS task definition in plaintext. IAM task execution role has secretsmanager:GetSecretValue permission scoped to this specific secret ARN only."

![ECS Task Definition Secrets](docx/Database%20Layer/Screenshot/image6.png)

---

### Criterion 5 - Encryption at Rest Verification

- Vào **RDS Console** → Chọn `minie-final-cluster` → Tab **Configuration**
- Trường **Encryption**: `Enabled`
- KMS Key: `aws/rds` (AWS Managed)

_(Chèn screenshot: RDS → Configuration → Encryption: Enabled)_

---

### Criterion 6 - Automated Backup Configuration

- Vào **RDS Console** → Chọn `minie-final-cluster` → Tab **Maintenance & Backups**
- Backup retention: **7 days**
- Backup window: `03:00–04:00 UTC`

![RDS Backup Configuration](docx/Database%20Layer/Screenshot/image7.png)

---

## 4) Working Query Evidence

### Query 1 - JOIN với 2 related tables

**Mục tiêu:** Lấy danh sách products kèm tên category (JOIN `products` + `categories`).

```sql
SELECT
    p.id,
    p.title,
    p.price,
    p.stock,
    c.name AS category_name
FROM products p
INNER JOIN categories c ON p.category_id = c.id
WHERE p.shop_id = 2
LIMIT 10;
```

![JOIN Query Results](docx/Database%20Layer/Screenshot/image8.png)

> **Note:** "This JOIN is served by the foreign key relationship `products.category_id → categories.id`. The query planner uses the primary key index on `categories.id` for the JOIN, avoiding full scan on categories table."

---

### Query 2 - Indexed Lookup

**Mục tiêu:** Lookup user theo email (indexed column) — được gọi mỗi lần login.

```sql
EXPLAIN SELECT id, email, role, password_hash
FROM users
WHERE email = 'nguyenvanhuyhoang2609@gmail.com';
```

**Kết quả EXPLAIN:** `type: const`, `key: users_email_uq`, `rows: 1`

![Indexed Lookup EXPLAIN Plan](docx/Database%20Layer/Screenshot/image9.png)

> **Note:** "Secondary index `idx_users_email` on `users.email` reduces login lookup from O(n) full table scan to O(log n) B-tree lookup. At 20 calls/min this matters at scale."

---
## Criterion 7: Bonus — Multi-AZ Failover Live Demo

> **Mục tiêu:** Chứng minh Multi-AZ hoạt động thật sự bằng cách trigger failover thủ công và verify qua AWS CLI.

### Bước 1: Verify cấu hình Multi-AZ trước khi demo

**AWS CLI — kiểm tra primary AZ và secondary AZ:**

```bash
aws rds describe-db-instances \
  --db-instance-identifier minie-final-cluste \
  --query 'DBInstances[0].{AZ:AvailabilityZone, SecondaryAZ:SecondaryAvailabilityZone, MultiAZ:MultiAZ}' \
  --region us-west-2
```

**Kết quả:**
```json
{
    "AZ": "us-west-2b",
    "SecondaryAZ": "us-west-2a",
    "MultiAZ": true
}
```

> Primary đang ở `us-west-2b` (sau failover), Standby ở `us-west-2a`. Multi-AZ cross-AZ thật sự.

![RDS Ceck primary and secondary](../Database%20Layer/Screenshot/image11.png)


---

### Bước 2: Trigger Failover

**RDS Console** → `minie-final-cluste` → **Actions** → **Reboot** → tick **"Reboot With Failover?"** → **Confirm**

![RDS Reboot with Failover](../Database%20Layer/Screenshot/image12.png)



---

### Bước 3: Verify qua Events Log

**RDS Console** → `minie-final-cluste` → Tab **Logs & events**

**Events ghi nhận (thực tế):**

| Thời gian | Event |
|---|---|
| Apr 24, 09:29 | The user requested a failover of the DB instance |
| Apr 24, 09:29 | **Multi-AZ instance failover started** |
| Apr 24, 09:29 | DB instance restarted |
| Apr 24, 09:29 | **Multi-AZ instance failover completed** ✅ |

![ RDS Events Log ](../Database%20Layer/Screenshot/image13.png)

---


## 5) Lambda + Bedrock Evidence

Tài liệu này minh chứng việc triển khai Knowledge Base sử dụng Amazon Bedrock để trả lời các câu hỏi dựa trên dữ liệu sản phẩm.

### Step 1 - S3 Bucket và Amazon Bedrock Knowledge Base

- **1.1. Chuẩn bị S3 Bucket:** Đây là nơi chứa các file text về sản phẩm của Database.
- **1.2. Tạo dữ liệu mẫu:** Tạo sẵn 3 file text (`.txt` hoặc `.json`) để test ghi thông tin 3 sản phẩm của bạn (ví dụ: `sp1.txt`, `sp2.txt`, `sp3.txt`) và nhấn Upload vào bucket S3 này.

  ![S3 Bucket Sample Data](docx/AI%20Bedrock%20Layer%20-%20Knowledge%20Base%20+%20Retrieval/Sceenshot/1_2.png)

- **1.3. Truy cập giao diện:** Truy cập vào giao diện quản lý **Amazon Bedrock** trên AWS Console.
- **1.4. Tạo Knowledge Base:** Chọn **Knowledge Base** > **Create Knowledge Base**, và chọn tùy chọn **Knowledge Base With Vector Store**.
- **1.5. Cấu hình Data Source:** Chọn Data source type là **S3**.
- **1.6. S3 URI:** Nhấn nút **Browse S3**, trỏ vào bucket đã tạo ở bước 1.1.
- **1.7. Embeddings Model:** Chọn mô hình **Titan Embeddings V2** hoặc **Titan Embeddings G1**.

  ![Embeddings Model Selection](docx/AI%20Bedrock%20Layer%20-%20Knowledge%20Base%20+%20Retrieval/Sceenshot/1_7.png)

- **1.8. Vector Store:** Chọn mục đầu tiên **Quick create a new vector store** (hoặc tạo sẵn trực tiếp từ S3).
- **1.9. Hoàn tất tạo:** Nhấn **Create**. Sau khi tạo xong, tiến hành **Sync** dữ liệu.
- **1.10. Test Knowledge Base:** Chọn model **Nova 2 Lite** để test. Kết quả cho thấy AI trả lời chính xác và có dẫn nguồn rõ ràng:

  ![Knowledge Base Test Results](docx/AI%20Bedrock%20Layer%20-%20Knowledge%20Base%20+%20Retrieval/Sceenshot/2_0.png)

---

### Step 2 - Chuẩn bị Layer (Thư viện MySQL)

Lambda mặc định không có thư viện để kết nối MySQL. Bạn phải tự tạo một "Layer" và upload lên.
**Các bước thực hiện trên máy tính cá nhân:**

- Tạo một thư mục tên là `nodejs`. (Bắt buộc phải tên là `nodejs`).
- Mở Terminal/CMD tại thư mục đó và chạy lệnh:
  ```bash
  npm install mysql2
  ```
- Nén thư mục `nodejs` này lại thành file `mysql-layer.zip`.
- Lên AWS Console -> Lambda -> Layers -> Create layer.
- Đặt tên là `mysql2-library`, upload file `.zip` lên và chọn Runtime là Node.js 20.x.

### Step 3 - Cấu hình IAM Role

Lambda cần "thẻ quyền lực" để nói chuyện với các dịch vụ khác. Tạo 1 Role mới cho Lambda với Policy JSON như sau:

![Lambda IAM Policy](docx/AI%20Bedrock%20Layer%20-%20Knowledge%20Base%20+%20Retrieval/Sceenshot/policy.jpg)

### Step 4 - Cấu hình Mạng (VPC & Security Group)

Đây là phần khó nhất, nếu sai Lambda sẽ bị Timeout.

#### A. Security Group (SG)

- **Lambda-SG:** Không cần Inbound, Outbound cho phép tất cả.
- **RDS-SG:** Thêm Inbound Rule cho cổng 3306, Source là ID của Lambda-SG.

#### B. VPC Endpoint

Vì Lambda nằm trong Private Subnet, nó cần "cổng đi tắt" để gọi Bedrock:

- Vào VPC -> Endpoints -> Create.
- Tìm Service: `com.amazonaws.us-west-2.bedrock-agent` (Lưu ý: phải có chữ agent).
- Chọn đúng VPC và Subnets mà Lambda đang dùng.
- Gán Security Group cho phép cổng 443 từ Lambda.

  ![VPC Endpoint Configuration](docx/AI%20Bedrock%20Layer%20-%20Knowledge%20Base%20+%20Retrieval/Sceenshot/vpc.jpg)

  ![alt text](<docx/AI Bedrock Layer - Knowledge Base + Retrieval/Sceenshot/sg.jpg>)

### Step 5 - Tạo Lambda & Cấu hình

- Tạo Lambda Function (Node.js 20.x).
- **Gán Layer:** Kéo xuống dưới cùng trang Lambda -> Add a layer -> Custom layers -> Chọn cái `mysql2-library` đã tạo ở Bước 2.
- **Cấu hình VPC:** Chọn đúng VPC, Subnets và Lambda-SG.
  > **Quan trọng:** Chọn `IPv4 only` ở phần IP address type.
- **Biến môi trường (Environment Variables):**

  ![Lambda Environment Variables](docx/AI%20Bedrock%20Layer%20-%20Knowledge%20Base%20+%20Retrieval/Sceenshot/env.jpg)
  DATA_SOURCE_ID: ID của Data Source (lấy trong Bedrock Console).

### Step 6 - Code logic xử lý (index.mjs)

```javascript
import mysql from "mysql2/promise";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import {
  BedrockAgentClient,
  StartIngestionJobCommand,
} from "@aws-sdk/client-bedrock-agent";

// Khởi tạo các Client bên ngoài handler để tái sử dụng (tăng hiệu năng)
const s3Client = new S3Client({});
const bedrockClient = new BedrockAgentClient({ region: "us-west-2" });

export const handler = async (event) => {
  let connection;
  try {
    console.log("Đang kết nối tới RDS...");
    connection = await mysql.createConnection({
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
    });

    // 1. Lấy dữ liệu sản phẩm từ bảng của dự án
    const [rows] = await connection.execute("SELECT * FROM products");

    // 2. Chuyển dữ liệu thành dạng văn bản để AI dễ đọc
    const dataContent = rows
      .map(
        (item) =>
          `Sản phẩm: ${item.title} | Giá: ${item.price} | Mô tả: ${item.description}`,
      )
      .join("\n");

    console.log("Đang upload dữ liệu lên S3...");
    // 3. Đẩy file lên S3 (tên key phải khớp với policy/dataSource)
    await s3Client.send(
      new PutObjectCommand({
        Bucket: process.env.S3_BUCKET,
        Key: "products_data.txt",
        Body: dataContent,
        ContentType: "text/plain",
      }),
    );

    console.log("Đang kích hoạt Bedrock Sync...");
    // 4. Ra lệnh cho Bedrock cập nhật kiến thức mới
    await bedrockClient.send(
      new StartIngestionJobCommand({
        knowledgeBaseId: process.env.KB_ID,
        dataSourceId: process.env.DATA_SOURCE_ID,
      }),
    );

    return {
      statusCode: 200,
      body: JSON.stringify({
        message: "Đồng bộ dữ liệu dự án mini_e thành công!",
      }),
    };
  } catch (error) {
    console.error("Lỗi chi tiết từ AWS:", error);
    return {
      statusCode: 500,
      body: JSON.stringify({
        msg: error.message || "Không có message",
        code: error.code || error.name || "Không có code",
        requestId: error.$metadata?.requestId || "N/A",
        stack: error.stack, // In stack trace để xác định chính xác dòng bị lỗi
      }),
    };
  } finally {
    if (connection) await connection.end();
  }
};
```

### Step 7 - Tự động hóa (Trigger)

Để hệ thống tự cập nhật dữ liệu mỗi ngày:

- Tại giao diện Lambda, nhấn Add trigger.
- Chọn EventBridge (CloudWatch Events).
- Tạo Rule mới, chọn Schedule expression.
- Nhập: `rate(1 day)` (Cứ 24h tự chạy 1 lần).

---

## 6) VPC + Networking Evidence

Trong phần này, nhóm sẽ trình bày về cấu trúc mạng đa tầng (Multi-Tier) và cách thức cấu hình bảo mật chuẩn giữa các tầng, cũng như phương thức tối ưu đường truyền dữ liệu với S3 Gateway Endpoint.

### 1. S3 Gateway Endpoint trong Route Table

![S3 Gateway Endpoint](docx/VPC%20và%20Networking%20-%20Multi-Tier%20+%20Gateway%20Endpoint/Screenshot/S3%20Gateway%20Endpoint.png)

**Notes:**
Nhóm đã thêm một S3 Gateway Endpoint vào route table của private subnet (định tuyến lưu lượng mạng cho S3 qua Prefix List `pl-68a54001` trực tiếp đến Target là `vpce-05a1f240aa0423a12`). Nhóm chọn dùng S3 Gateway Endpoint thay vì định tuyến qua NAT Gateway vì giải pháp này giữ mọi dữ liệu giao tiếp với S3 nằm hoàn toàn trong mạng nội bộ của AWS. Điều này không chỉ tăng cường bảo mật (vì không đi ra Internet public) mà còn loại bỏ hoàn toàn chi phí xử lý dữ liệu (data processing charges) của NAT Gateway.

---

### 2. Database Security Group Inbound Rule

![Database Inbound rules](docx/VPC%20và%20Networking%20-%20Multi-Tier%20+%20Gateway%20Endpoint/Screenshot/Database%20Inbound%20rules.png)

**Notes:**
Đối với Security Group của tầng Database, nhóm cấu hình Inbound Rules chỉ chấp nhận kết nối tại cổng của database với Nguồn (Source) là các Security Groups của tầng Application (ví dụ: ECS và Lambda SGs). Nhóm ưu tiên việc tham chiếu ID Security Group (Security Group referencing) thay vì dùng dải IP (subnet CIDRs) vì điều này tuân thủ tuyệt đối nguyên tắc quyền hạn tối thiểu (least privilege). Nó cũng tự động mở rộng (scales automatically): bất kỳ application instance mới nào được tạo ra và gắn App SG sẽ lập tức có quyền kết nối tới database mà không cần phải đi cập nhật dải IP theo cách thủ công.

---

### 3. Application Tier Security Groups (ECS & Lambda)

![ECS Inbound rules](docx/VPC%20và%20Networking%20-%20Multi-Tier%20+%20Gateway%20Endpoint/Screenshot/ECS%20Inbound%20rules.png)

![Lambda Inbound rules](docx/VPC%20và%20Networking%20-%20Multi-Tier%20+%20Gateway%20Endpoint/Screenshot/Lambda%20Inbound%20rules.png)

**Notes:**
Tầng Application của nhóm gồm các tài nguyên tính toán như ECS và Lambda. Các Security Group của chúng đóng vai trò là lớp bảo mật trung gian. Ví dụ, ECS được cấu hình để chỉ nhận traffic đầu vào từ tầng Public (ALB Security Group). Điều này đảm bảo không có kết nối trực tiếp nào từ ngoài Internet có thể chọc vào các server ứng dụng của nhóm.

---

### 4. Public Tier Security Group (ALB)

![ALB Inbound rules](docx/VPC%20và%20Networking%20-%20Multi-Tier%20+%20Gateway%20Endpoint/Screenshot/ALB%20Inbound%20rules.png)

**Notes:**
Application Load Balancer (ALB) nằm ở tầng public. Security Group của nó được cấu hình để cho phép traffic HTTP/HTTPS vào từ ngoài Internet (`0.0.0.0/0`). Nó đóng vai trò là cổng vào public duy nhất để định tuyến các request đến với tầng private application của nhóm.

## 7) Negative Security Test

### 1. Scenario 1 – Kiểm thử truy cập trực tiếp vào ECS backend (Bypass ALB)

### 1.1 Mục tiêu

Xác minh rằng dịch vụ backend chạy trên ECS không thể bị truy cập trực tiếp từ các nguồn không được phép, và chỉ chấp nhận lưu lượng đi qua lớp phân phối/phía trước theo đúng thiết kế bảo mật.

### 1.2 Bối cảnh kiến trúc

Trong kiến trúc hiện tại:

- Frontend công khai được phân phối qua domain:
  `https://minie-ecommercehoangdeptraisieucaovutru.software`
- Backend ECS được đăng ký trong target group:
  `10.0.0.149:3000` (Healthy)
- Backend là tài nguyên nội bộ trong VPC, không được expose trực tiếp ra Internet

Mục tiêu của kiểm thử là chứng minh rằng:

- người dùng/public chỉ truy cập được qua lớp công khai
- không thể truy cập trực tiếp vào private backend IP

### 1.3 Kiểm thử từ Internet qua domain công khai

**Dẫn chứng:**

![Scenario 1 - Internet Access Test](docx/Negative%20Security%20Test/scenario-1/scenario-1.png)

**Nguồn:** Kali Linux ngoài Internet

**Lệnh thực hiện:**

```bash
curl -i https://minie-ecommercehoangdeptraisieucaovutru.software/api/categories
```

**Kết quả:**

- Trả về `HTTP/2 200`
- Response body là HTML frontend
- Header cho thấy response đi qua `CloudFront` và `AmazonS3`

**Nhận xét:**

- Domain công khai hoạt động bình thường
- Người dùng Internet có thể truy cập lớp public của hệ thống
- Route `/api/categories` trên domain công khai hiện tại đang được xử lý ở lớp frontend phân phối qua CloudFront/S3, không truy cập trực tiếp vào backend ECS

### 1.4 Kiểm thử từ EC2 nội bộ qua domain công khai (Lần 1)

**Dẫn chứng:**

![Scenario 1 - EC2 Internal Access Test](docx/Negative%20Security%20Test/scenario-1/scenario-2.png)

**Nguồn:** EC2 nội bộ `10.0.0.10`

**Lệnh thực hiện:**

```bash
curl -i https://minie-ecommercehoangdeptraisieucaovutru.software/api/categories
```

**Kết quả:**

- Trả về `HTTP/2 200`
- Response body là HTML frontend
- Header cho thấy response đi qua `CloudFront` và `AmazonS3`

**Nhận xét:**

- Domain công khai hoạt động bình thường
- Người dùng Internet có thể truy cập lớp public của hệ thống
- Route `/api/categories` trên domain công khai hiện tại đang được xử lý ở lớp frontend phân phối qua CloudFront/S3, không truy cập trực tiếp vào backend ECS

### 1.4 Kiểm thử từ EC2 nội bộ qua domain công khai

**Nguồn:** EC2 nội bộ `10.0.0.10`

**Lệnh thực hiện:**

```bash
curl -i https://minie-ecommercehoangdeptraisieucaovutru.software/api/categories
```

**Kết quả:**

- Trả về `HTTP/2 200`
- Response body là HTML frontend
- Header tiếp tục cho thấy response đi qua `CloudFront` và `AmazonS3`

**Nhận xét:**

- Cả từ Internet và từ EC2 nội bộ, domain công khai đều chỉ trả về lớp frontend public
- Điều này cho thấy domain public không cung cấp đường truy cập trực tiếp đến ECS backend

### 1.5 Kiểm thử truy cập trực tiếp vào backend ECS

**Dẫn chứng:**

![Scenario 1 - Direct Backend Access Test](docx/Negative%20Security%20Test/scenario-1/scenario-3.png)

**Nguồn:** EC2 nội bộ `10.0.0.10`

**Lệnh thực hiện:**

```bash
curl -i http://10.0.0.149:3000/api/categories
```

**Kết quả:**

```text
curl: (28) Failed to connect to 10.0.0.149 port 3000: Couldn't connect to server
```

**Nhận xét:**

- Dù request xuất phát từ bên trong VPC, kết nối trực tiếp tới backend vẫn thất bại
- Điều này cho thấy backend không chấp nhận truy cập trực tiếp từ nguồn không được cho phép
- Backend chỉ nên nhận traffic từ thành phần trung gian hợp lệ theo thiết kế, thay vì từ các máy bất kỳ trong VPC

### 1.6 Phân tích bảo mật

Scenario này chứng minh ba điểm quan trọng:

1. **Lớp public và lớp backend được tách biệt**
   - Domain public chỉ phục vụ frontend public
   - Backend ECS không bị lộ trực tiếp

2. **Không thể bypass lớp phân phối công khai để gọi thẳng backend**
   - Truy cập trực tiếp vào `10.0.0.149:3000` thất bại
   - Điều này giúp giảm nguy cơ bypass các lớp kiểm soát ở phía trước

3. **Hạn chế lateral movement trong nội bộ**
   - Ngay cả EC2 nội bộ cũng không thể tùy ý gọi trực tiếp tới backend
   - Đây là dấu hiệu tốt của việc áp dụng nguyên tắc least privilege

### 1.7 Kết luận

Kết quả kiểm thử cho thấy:

- Truy cập qua domain công khai → **thành công ở lớp frontend public**
- Truy cập trực tiếp tới ECS backend → **bị từ chối**

Điều này xác nhận rằng hệ thống đã cô lập backend đúng cách và ngăn chặn hành vi bypass lớp phân phối/phía trước để truy cập trực tiếp vào dịch vụ ứng dụng nội bộ.

---

### 2. Scenario 2 – Kiểm thử kiểm soát truy cập tới RDS MySQL

### 2.1 Mục tiêu

Xác minh rằng Amazon RDS MySQL chỉ cho phép truy cập từ các nguồn nội bộ đã được cấp quyền thông qua Security Group, và từ chối truy cập từ Internet cũng như từ các tài nguyên nội bộ không được phép.

### 2.2 Bối cảnh kiến trúc

- **DB identifier**: `minie-final-cluste`
- **Endpoint**: `minie-final-cluste.cl46ao2sg9te.us-west-2.rds.amazonaws.com`
- **Port**: `3306`
- **VPC**: `vpc-00d67111bf0b4f3c6`
- **Publicly accessible**: **No**
- **Security Group**: `db-sg (sg-07e4be3b8955b1944)`

Inbound của `db-sg` chỉ cho phép từ:

- `sg-00e07d33b9e1ad37d`
- `sg-0a7416a278f80c9c9`

### 2.3 Kiểm thử không hợp lệ từ Internet (External Negative Test)

**Dẫn chứng:**

![Scenario 2 - External Negative Test](docx/Negative%20Security%20Test/scenario-2/scenario-1.png)

**Nguồn:** Kali Linux (ngoài VPC)

**Lệnh:**

```bash
nc -vz minie-final-cluste.cl46ao2sg9te.us-west-2.rds.amazonaws.com 3306
```

**Kết quả:**

```text
minie-final-cluste.cl46ao2sg9te.us-west-2.rds.amazonaws.com [10.0.0.164] 3306 (mysql) : Connection refused
```

**Nhận xét:**

- RDS resolve về IP private `10.0.0.164`
- Không thể thiết lập kết nối từ Internet
- Database không bị expose public

### 2.4 Kiểm thử không hợp lệ trong nội bộ (Internal Negative Test)

**Dẫn chứng:**

![Scenario 2 - Internal Negative Test](docx/Negative%20Security%20Test/scenario-2/scenario-2.png)

**Nguồn:** EC2 `10.0.0.10` (không thuộc SG được phép)

**Lệnh:**

```bash
nc -vz minie-final-cluste.cl46ao2sg9te.us-west-2.rds.amazonaws.com 3306
```

**Kết quả:**

```text
nc: connect to ... port 3306 (tcp) failed: Connection timed out
```

**Nhận xét:**

- EC2 nằm trong cùng VPC vẫn không thể truy cập DB
- Kết nối bị timeout → bị chặn bởi Security Group
- Chứng minh kiểm soát truy cập nội bộ đang hoạt động đúng

### 2.5 Kiểm thử hợp lệ từ nguồn được cấp quyền (Positive Test)

**Dẫn chứng:**

![Scenario 2 - Positive Test](docx/Negative%20Security%20Test/scenario-2/scenario-3.jpg)

**Nguồn:** EC2/ECS thuộc Security Group được phép

**Lệnh:**

```bash
mysql -h minie-final-cluste.cl46ao2sg9te.us-west-2.rds.amazonaws.com -u admin -p
```

**Kết quả:**

```text
Welcome to the MariaDB monitor.
Your MySQL connection id is 174
Server version: 8.4.8
```

**Nhận xét:**

- Kết nối tới database thành công
- Xác nhận Security Group đã cho phép đúng nguồn hợp lệ
- Database hoạt động bình thường

### 2.6 Phân tích bảo mật

Kết quả kiểm thử cho thấy ba lớp bảo vệ:

1. **Cô lập khỏi Internet**
   - RDS không public (`Publicly accessible = No`)
   - Không thể truy cập từ bên ngoài

2. **Kiểm soát nội bộ theo Security Group**
   - Chỉ các SG cụ thể mới được phép truy cập port `3306`
   - Không mở rộng theo CIDR

3. **Ngăn chặn lateral movement**
   - EC2 nội bộ không được phép vẫn bị từ chối
   - Giảm rủi ro tấn công nội bộ

### 2.7 Kết luận

- Internet → DB → ❌ bị từ chối
- EC2 không được phép → DB → ❌ bị từ chối
- EC2/ECS được phép → DB → ✅ thành công

Điều này xác nhận rằng RDS MySQL được cấu hình đúng theo nguyên tắc **least privilege** và đảm bảo bảo mật cho lớp dữ liệu.

---

### 3. Scenario 3 – Kiểm thử phát hiện fuzzing/thăm dò tài nguyên bằng FFUF và AWS WAF

### 3.1 Mục tiêu

Xác minh rằng hệ thống có khả năng phát hiện và chặn các hành vi thăm dò tài nguyên web (content discovery / fuzzing) bằng công cụ tự động, cụ thể là **FFUF**, thông qua AWS WAF.

### 3.2 Bối cảnh kiểm thử

Trong quá trình kiểm thử bảo mật ứng dụng web, nhóm sử dụng công cụ **FFUF (Fuzz Faster U Fool)** để dò tìm các đường dẫn và tài nguyên có thể truy cập trên domain công khai:

- `https://minie-ecommercehoangdeptraisieucaovutru.software`

Mục tiêu của kiểm thử này là đánh giá xem hệ thống có cơ chế phát hiện hành vi quét tự động hay không, đặc biệt dựa trên đặc điểm nhận dạng của request như **User-Agent**.

### 3.3 Script kiểm thử sử dụng

```bash
#!/bin/bash
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
TARGET="https://minie-ecommercehoangdeptraisieucaovutru.software"

echo "[*] FFUF Fuzz — $TARGET"

ffuf -u "$TARGET/FUZZ" \
  -w /usr/share/wordlists/dirb/common.txt \
  -fs 475 \
  -mc all \
  -rate 10 \
  -c -v
```

### 3.4 Giải thích cấu hình kiểm thử

- `-u "$TARGET/FUZZ"`: thay thế từ khóa `FUZZ` bằng từng mục trong wordlist để dò đường dẫn
- `-w /usr/share/wordlists/dirb/common.txt`: sử dụng wordlist phổ biến để tìm thư mục/file
- `-fs 475`: bỏ qua các response có kích thước 475 byte (trong hệ thống này thường là trang HTML frontend mặc định)
- `-mc all`: hiển thị mọi mã trạng thái HTTP
- `-rate 10`: giới hạn tốc độ 10 request/giây để tránh tạo tải quá cao
- `-c -v`: hiển thị output có màu và chi tiết hơn

Kiểm thử này mô phỏng hành vi của một attacker đang cố gắng tìm các endpoint ẩn hoặc tài nguyên chưa được công bố công khai.

### 3.5 Kết quả từ AWS WAF

**Dẫn chứng:**

![WAF FFUF Detection](docx/Negative%20Security%20Test/waf/fuff/fuff-1.png)

![WAF FFUF Blocked Request](docx/Negative%20Security%20Test/waf/fuff/ffuf-2.png)

AWS WAF ghi nhận và chặn request với thông tin như sau:

- **Source IP**: `1.54.91.128`
- **Rule inside rule group**:
  `AWS#AWSManagedRulesCommonRuleSet#UserAgent_BadBots_HEADER`
- **Action**: `BLOCK`
- **Time**: `Fri Apr 24 2026 00:00:57 GMT+0700`
- **URI**: `/feedback`

**Request bị chặn:**

```http
GET /feedback HTTP/1.1
Host: minie-ecommercehoangdeptraisieucaovutru.software
User-Agent: Fuzz Faster U Fool v2.1.0-dev
Accept-Encoding: gzip
```

### 3.6 Phân tích kết quả

Kết quả trên cho thấy AWS WAF đã phát hiện request xuất phát từ công cụ FFUF dựa trên **User-Agent**:

- `Fuzz Faster U Fool v2.1.0-dev`

Rule được kích hoạt là:

- `AWSManagedRulesCommonRuleSet`
- `UserAgent_BadBots_HEADER`

Điều này chứng minh rằng hệ thống không chỉ kiểm tra payload độc hại, mà còn có khả năng phát hiện các dấu hiệu của **bot scanning / fuzzing tool**.

WAF đã thực hiện hành động:

- **BLOCK** request trước khi request này đi sâu hơn vào hệ thống

### 3.7 Ý nghĩa bảo mật

Scenario này chứng minh các điểm sau:

1. **Hệ thống có khả năng phát hiện công cụ dò quét tự động**
   - FFUF bị nhận diện qua User-Agent
   - Request bị chặn ở lớp WAF

2. **Giảm nguy cơ content discovery trái phép**
   - Attacker khó dò ra các endpoint ẩn bằng công cụ phổ biến

3. **Bảo vệ lớp ứng dụng trước reconnaissance**
   - Giai đoạn do thám (reconnaissance) thường là bước đầu trong chuỗi tấn công
   - Việc chặn sớm giúp giảm khả năng tiếp tục khai thác

4. **Managed rules của AWS WAF đang hoạt động hiệu quả**
   - Không cần tự viết custom rule phức tạp nhưng vẫn chặn được hành vi phổ biến

### 3.8 Hạn chế của cơ chế này

Mặc dù WAF đã chặn được FFUF trong kiểm thử này, vẫn cần lưu ý:

- Việc chặn ở đây chủ yếu dựa trên **User-Agent**
- Nếu attacker thay đổi User-Agent thành trình duyệt thông thường, request có thể không còn bị rule này phát hiện
- Do đó, cần kết hợp thêm:
  - rate limiting
  - bot control
  - custom rules
  - behavioral detection
  - logging và monitoring liên tục

### 3.9 Kết luận

Kết quả kiểm thử cho thấy AWS WAF đã phát hiện và chặn thành công request fuzzing được gửi bởi công cụ FFUF thông qua rule `UserAgent_BadBots_HEADER`.

Điều này xác nhận rằng hệ thống có lớp phòng vệ hiệu quả ở tầng ứng dụng, có khả năng nhận diện và ngăn chặn các hành vi quét tài nguyên tự động trước khi chúng tiếp cận sâu hơn vào backend.

---

### 4. Scenario 4 – Kiểm thử nâng cao XSS với AWS WAF

### 4.1 Mục tiêu

Đánh giá khả năng phát hiện và ngăn chặn tấn công Cross-Site Scripting (XSS) của AWS WAF thông qua việc mô phỏng nhiều payload khác nhau (từ cơ bản đến nâng cao), đồng thời kiểm tra khả năng bypass rule.

### 4.2 Bối cảnh kiến trúc

Hệ thống sử dụng:

- AWS WAF (gắn với CloudFront/ALB)
- Managed Rule Group:
  - `AWSManagedRulesCommonRuleSet`

- Rule:
  - `CrossSiteScripting_QUERYARGUMENTS`

WAF được cấu hình để phân tích:

- Query string
- HTTP headers
- Request body

### 4.3 Bằng chứng từ WAF (Sampled Requests)

**Dẫn chứng:**

![WAF XSS Detection 1](docx/Negative%20Security%20Test/waf/xss/xss-1.png)

![WAF XSS Detection 2](docx/Negative%20Security%20Test/waf/xss/xss-2.png)

![WAF XSS Detection 3](docx/Negative%20Security%20Test/waf/xss/xss-3.png)

AWS WAF ghi nhận request bị chặn:

```text
Rule: AWSManagedRulesCommonRuleSet#CrossSiteScripting_QUERYARGUMENTS
Action: BLOCK
URI:
/api/users?q=%3Cbody%20onload%3Dalert(document.domain)%3E
```

**Request:**

```http
GET /api/users?q=<body onload=alert(document.domain)>
User-Agent: Mozilla/5.0 Chrome/143.0.0.0
```

Điểm quan trọng:

- Không dùng User-Agent của tool (giống browser thật)
- Vẫn bị block

### 4.4 Phương pháp kiểm thử (Automation Script)

Nhóm sử dụng script bash để:

1. Login lấy JWT token
2. Gửi request có Authorization header
3. Test nhiều endpoint:
   - `/api/orders`
   - `/api/products`
   - `/api/users`

4. Test nhiều payload XSS:
   - cơ bản
   - HTML event
   - SVG
   - iframe
   - bypass attempt

### 4.5 Nhóm payload kiểm thử

| Nhóm        | Ví dụ                      |
| ----------- | -------------------------- |
| Basic       | `<script>alert()</script>` |
| Event-based | `<img onerror=alert()>`    |
| SVG         | `<svg/onload=alert()>`     |
| DOM-based   | `<body onload=...>`        |
| Advanced    | `<details ontoggle=...>`   |
| Bypass      | encoding, nested script    |

### 4.6 Kết quả kiểm thử

Script thực hiện:

- GET query test (`q=`, `search=`)
- POST body test

Kết quả tổng hợp:

```text
Blocked: X/Y
Bypass: Z/Y
WAF Score: ~%
```

Trong log thực tế:

- Payload `<body onload=alert(document.domain)>`
- Bị WAF block (HTTP 403)

### 4.7 Phân tích kết quả

#### 1. WAF phát hiện payload thực, không chỉ signature đơn giản

- Payload không phải `<script>`
- vẫn bị block

Điều này cho thấy WAF có logic phân tích theo mẫu hành vi, không chỉ phụ thuộc vào một chuỗi ký tự cố định.

#### 2. Không phụ thuộc User-Agent

- Request dùng:
  `Mozilla/5.0 Chrome/143`
- vẫn bị block

Điều này cho thấy WAF không chỉ dựa vào nhận diện tool mà phân tích trực tiếp nội dung payload.

#### 3. Bảo vệ cả authenticated endpoint

- Request có:
  `Authorization: Bearer <token>`
- vẫn bị block

Điều này chứng minh WAF vẫn kiểm tra cả lưu lượng sau đăng nhập.

#### 4. Phân tích đa layer (GET + POST)

- Query string → bị block
- Body (POST) → bị block

Điều này cho thấy WAF không chỉ lọc URL mà còn có khả năng kiểm tra payload trong nội dung request.

#### 5. Khả năng chống bypass

Một số payload thử bypass:

```html
<scr<script>ipt>alert(1)</scr</script>ipt>
<IMG SRC=1 ONERROR=&#X61;&#X6C;&#X65;&#X72;&#X74;(1)>
```

Nếu các payload này bị block, điều đó cho thấy WAF có cơ chế decode và normalize dữ liệu trước khi đối sánh rule.

### 4.8 Ý nghĩa bảo mật

Scenario này chứng minh hệ thống có:

1. **Protection ở application layer**
   - không chỉ network layer

2. **Detection thông minh**
   - không phụ thuộc tool signature
   - phát hiện payload thật

3. **Chống tấn công real-world**
   - reflected XSS
   - stored XSS
   - DOM-based XSS

4. **Giảm nguy cơ account takeover**
   - vì token/session có thể bị đánh cắp qua XSS

### 4.9 Hạn chế và đề xuất cải tiến

Dù WAF hoạt động tốt, vẫn có rủi ro:

- Payload obfuscation nâng cao có thể bypass
- Encoding nhiều lớp
- JavaScript context-based injection

Đề xuất:

- Bổ sung:
  - AWS WAF Bot Control
  - Rate limiting
  - Custom regex rule

- Kết hợp:
  - validation phía backend
  - output encoding phía frontend

### 4.10 Kết luận

Kết quả kiểm thử cho thấy:

- WAF đã chặn thành công nhiều payload XSS
- Hoạt động ngay cả với request hợp lệ đã đăng nhập
- Không phụ thuộc User-Agent
- Phân tích cả query và request body

Điều này xác nhận hệ thống có khả năng bảo vệ mạnh ở tầng ứng dụng trước các tấn công XSS trong thực tế.

---

## 8) Bonus (Optional)

### Scenario - [ten scenario]

- Hiện tại chưa có

---

## 9) Final Checklist Before Submit

- [ ] **Cover**: Đủ group number, members, database engine+paradigm, link W2
- [ ] **Data Access Pattern Log**: Đủ Parts A, B, C (với reasoning và tradeoff)
- [ ] **Deployment Evidence**: Mỗi criterion có 1 screenshot/CLI output + rationale rõ ràng
- [ ] **Working Query Evidence**: 2 queries với real results + screenshots + why representative
- [ ] **Lambda + Bedrock**: CloudWatch logs + Bedrock response (không Playground)
- [ ] **VPC + Networking**: S3 Gateway Endpoint + DB Security Group inbound rule
- [ ] **Negative Security Test**: Ít nhất 1 unauthorized access bị denied
- [ ] **All Links & Images**: Accessible, không missing, đường dẫn đúng cấu trúc folder
