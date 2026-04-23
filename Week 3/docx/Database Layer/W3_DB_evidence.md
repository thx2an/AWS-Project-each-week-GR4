# Evidence Pack — W3: Xương Sống Database
**Project:** Mini-E (E-commerce Full-stack)
**Database path:** Amazon Aurora MySQL / Relational
**Link W2 Evidence:** *(chèn link commit W2 tại đây)*

---

## Section 1: Cover & Infrastructure Summary

| Thành phần | Chi tiết |
|---|---|
| **VPC** | 3-tier: Public (ALB) → Private App (ECS Fargate) → Private DB (Aurora) |
| **Compute** | ECS Fargate — NestJS Backend (`minie-final-task`) |
| **Frontend** | S3 (`minie-frontend-final`) + CloudFront CDN |
| **Database** | Amazon RDS MySQL Community 8.x — Instance `minie-final-cluste` |
| **Region** | `us-west-2` (Oregon) |
| **AZ** | `us-west-2a` (single AZ — SPOF acknowledged, see Section 3) |
| **DB Endpoint** | `minie-final-cluste.ci46ao2sg9te.us-west-2.rds.amazonaws.com` |

---

## Section 2: Data Access Pattern Log

### Part A — 3 Access Patterns Thật Từ App

1. **User Authentication** — Lookup user theo `email` để verify credentials và issue JWT token.
   - Frequency: ~20 calls/phút lúc peak.

2. **Product Catalog** — Fetch danh sách products JOIN với `categories` để lấy category name, lọc theo `shop_id` hoặc `slug`.
   - Frequency: ~100 calls/phút lúc peak.

3. **Order Processing** — Tạo order + decrement `products.stock` trong 1 ACID transaction đảm bảo inventory không bao giờ âm.
   - Frequency: ~10 calls/phút lúc peak.

---

### Part B — Engine + Paradigm + Reasoning

- **Paradigm chọn:** Relational
- **Engine:** Amazon Aurora MySQL 8.x (Managed)

**Tại sao Relational fit:**
- App có foreign key relationships: `users` → `orders` → `order_items` → `products` → `categories`. JOIN là bắt buộc, không thể tránh.
- ACID transactions cần thiết cho payment (VNPAY) và order processing: nếu decrement stock thành công nhưng insert order fail → phải rollback toàn bộ.
- Complex queries: lọc sản phẩm theo category, price range, sắp xếp theo rating — đây là use case điển hình của relational engine.

**Index đã tạo:**
- `users.email` — INDEX để lookup O(log N) thay vì full scan.
- `products.slug` — UNIQUE INDEX cho SEO-friendly URL lookup.
- `order_items.order_id` — INDEX hỗ trợ JOIN giữa `orders` và `order_items`.

**HA/Backup Plan:**
- Multi-AZ enabled trên Aurora cluster (writer + reader instance).
- Automated backup: 7 ngày retention, daily snapshot lúc 03:00 UTC.
- Point-in-time restore available.

**Tại sao chọn RDS MySQL Community:**
- Cost-effective cho giai đoạn học: `db.t3.micro` nằm trong AWS Free Tier.
- MySQL 8.x compatible hoàn toàn với TypeORM driver trong NestJS — không cần sửa code.
- Managed service: AWS lo OS patching, storage management, automated backup.

**SPOF Acknowledgment (Single-AZ):**
- Hiện tại instance chạy trên 1 AZ duy nhất (`us-west-2a`) — đây là single point of failure được chấp nhận cho môi trường học.
- **Mitigation plan:** Automated backup 7 ngày retention cho phép restore nếu instance fail. Trong production thật, sẽ enable Multi-AZ để tự động failover sang AZ thứ 2 (~60-120 giây).

---

<!-- ### Part C — "Wrong-Paradigm" Test

**Pattern thử nghiệm:** Order Processing (tạo order + decrement inventory).

**Nếu dùng Key-Value (DynamoDB):**
- Phải dùng `TransactWriteItems` qua tối thiểu 3 items (orders table, order_items table, products table) — DynamoDB transactions có giới hạn 100 items/transaction và cost cao hơn.
- Schema cứng: mỗi item thêm attributes khác nhau dẫn đến sparse items, tốn WCU (Write Capacity Unit) lãng phí.
- **Failure point:** Đảm bảo `stock >= 0` trong flash sale cần Optimistic Locking (condition expression) cho từng item riêng lẻ — nếu 1000 users cùng lúc mua sản phẩm cuối cùng, DynamoDB conditional writes sẽ fail 999 request và cần retry loop phức tạp ở application level. Relational engine xử lý việc này tự nhiên qua row-level locking + `SELECT ... FOR UPDATE`.

--- -->

## Section 3: Deployment Evidence — Database Layer

### Bước 1: Chọn Subnet Group (Private Database Tier)

**Cấu hình:** RDS instance `minie-final-cluste` được đặt trong **private subnet** — không có route ra internet, Publicly accessible: **No**.

- Subnet Group Name: `minie-db-subnet-group`
- Subnets được chọn:
  - `DB-Subnet-1` — `10.0.2.0/24` — AZ `us-west-2a`
  - `DB-Subnet-2` — `10.0.3.0/24` — AZ `us-west-2b`
- Không có Internet Gateway route trong route table của 2 subnet này.

> **Config note:** "We placed the RDS instance in a private subnet with no IGW route and Publicly Accessible = No. Only the ECS application tier can reach port 3306 via the db-sg security group rule."

![alt text](/Week%203/docx/Database%20Layer/Screenshot/image.png)

---

### Bước 2: Security Group cho Database (sg-rds)

**Cấu hình Inbound Rule:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| MySQL/Aurora | TCP | 3306 | `sg-ecs` (Security Group ID của ECS) |

> **Config note:** "Inbound rule references the ECS Security Group ID directly — not a CIDR block. This means only tasks running in the ECS service can connect to Aurora. Even if someone gains access to the VPC, they cannot reach the DB without being in sg-ecs."

![alt text](/Week%203/docx/Database%20Layer/Screenshot/image1.png)


---

### Bước 3: Tạo RDS MySQL Instance

**Thông số cấu hình:**

| Parameter | Giá trị | Lý do |
|---|---|---|
| Engine | RDS MySQL Community 8.x | Compatible TypeORM/MySQL driver, AWS managed |
| Template | Free Tier | Phù hợp môi trường học, cost thấp |
| DB Instance ID | `minie-final-cluste` | |
| DB Instance Class | `db.t3.micro` | Free Tier eligible, đủ cho workload học |
| Multi-AZ | **Disabled** — SPOF acknowledged | Single AZ (us-west-2a), chấp nhận cho môi trường học |
| DB Name | `minie_db` | |
| Encryption at Rest | **Enabled** | Bảo mật data lưu trên disk |
| Backup Retention | 7 ngày | Đủ để point-in-time restore thay cho Multi-AZ |
| Public Access | **No** (Internet access gateway: Disabled) | Chỉ reach được từ trong VPC |
| VPC | Mini-F-Final-vpc | |
| Subnet Group | `minie-db-subnet-group` | Private subnets |
| Security Group | `db-sg` (sg-07e4bc33a8955b1944) | Chỉ cho phép từ ECS security group |

> **Config note:** "We chose db.t3.micro (Free Tier) over a larger instance because our current workload does not require high memory. Multi-AZ is disabled — we acknowledge this as a SPOF for this week's learning environment. The mitigation is 7-day automated backup with point-in-time restore capability. In production, we would enable Multi-AZ for automatic failover."


![alt text](/Week%203/docx/Database%20Layer/Screenshot/image2.png)
![alt text](/Week%203/docx/Database%20Layer/Screenshot/image3.png)
![alt text](/Week%203/docx/Database%20Layer/Screenshot/image4.png)
![alt text](/Week%203/docx/Database%20Layer/Screenshot/image5.png)


---

### Bước 4: Kết Nối ECS Backend → RDS Instance
Credentials được lưu trong **AWS Secrets Manager** (không hardcode trong code):

| Secret Key | Giá trị (ẩn) |
|---|---|
| `DB_HOST` | `minie-final-cluste.ci46ao2sg9te.us-west-2.rds.amazonaws.com` |
| `DB_PORT` | `3306` |
| `DB_USER` | `admin` |
| `DB_PASS` | `********` |
| `DB_NAME` | `minie_db` |

ECS Task Definition inject secrets từ Secrets Manager vào container environment variable thông qua `valueFrom` ARN — không có plain-text credential trong code hay console.

> **Config note:** "We use AWS Secrets Manager valueFrom injection so that credentials are never stored in the Docker image or ECS task definition in plaintext. IAM task execution role has secretsmanager:GetSecretValue permission scoped to this specific secret ARN only."

![alt text](/Week%203/docx/Database%20Layer/Screenshot/image6.png)

---

### Bước 5: Verify Encryption At Rest

- Vào **RDS Console** → Chọn `minie-final-cluster` → Tab **Configuration**
- Trường **Encryption**: `Enabled`
- KMS Key: `aws/rds` (AWS Managed)

*(Chèn screenshot: RDS → Configuration → Encryption: Enabled)*

---

### Bước 6: Verify Automated Backup

- Vào **RDS Console** → Chọn `minie-final-cluster` → Tab **Maintenance & Backups**
- Backup retention: **7 days**
- Backup window: `03:00–04:00 UTC`

![alt text](/Week%203/docx/Database%20Layer/Screenshot/image7.png)


---

## Section 4: Working Query Evidence

### Query 1 — JOIN Query (2 related tables)

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
![alt text](/Week%203/docx/Database%20Layer/Screenshot/image8.png)



> **Note:** "This JOIN is served by the foreign key relationship `products.category_id → categories.id`. The query planner uses the primary key index on `categories.id` for the JOIN, avoiding full scan on categories table."

---

### Query 2 — Indexed Lookup

**Mục tiêu:** Lookup user theo email (indexed column) — được gọi mỗi lần login.

```sql
EXPLAIN SELECT id, email, role, password_hash
FROM users
WHERE email = 'nguyenvanhuyhoang2609@gmail.com';
```

**Kết quả EXPLAIN:** `type: const`, `key: users_email_uq`, `rows: 1`

![alt text](/Week%203/docx/Database%20Layer/Screenshot/image9.png)

> **Note:** "Secondary index `idx_users_email` on `users.email` reduces login lookup from O(n) full table scan to O(log n) B-tree lookup. At 20 calls/min this matters at scale."

---

## Section 7: Negative Security Test

**Test:** Thử kết nối trực tiếp vào RDS endpoint từ ngoài VPC (internet).

**Command thử từ máy local:**
```bash
mysql -h minie-final-cluste.cl46ao2sg9te.us-west-2.rds.amazonaws.com \
      -u admin -p minie_db
```

**Kết quả:** Connection timeout — `ERROR 2003 (HY000): Can't connect to MySQL server ... (timed out)`

> **Note:** "Connection attempt from public internet is blocked because: (1) RDS instance has Publicly Accessible = No, (2) Security Group sg-rds only allows inbound 3306 from sg-bastion/sg-ecs, (3) No route from internet to private subnet. All 3 layers independently block the attempt."
![alt text](/Week%203/docx/Database%20Layer/Screenshot/image10.png)

