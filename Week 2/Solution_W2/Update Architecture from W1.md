## GÓP Ý SỬA/THÊM CHO KIẾN TRÚC DIAGRAM TUẦN 2
Dựa trên hình 3-tier.png hiện tại, chúng ta sẽ có nhưng solution như sau:

1. Thiếu kho lưu trữ S3 (Bắt buộc phải có ít nhất 2 bucket)

* Vấn đề: Hiện tại bạn đang dùng Amplify để host Frontend và chưa có kho lưu trữ file do user tải lên hay backup. Yêu cầu tuần 2 bắt buộc phải chỉ đích danh các S3.
Cách vẽ thêm:
Thêm icon Amazon S3 số 1 (Ví dụ tên: Static Assets / Frontend Bucket). Nếu Amplify dùng S3 ẩn bên dưới, hãy vẽ rõ ra, hoặc chuyển host Web tĩnh qua S3 nối với CloudFront. Kèm chú thích: Storage Class: S3 Standard.
Thêm icon Amazon S3 số 2 (Ví dụ tên: User Uploads / Media Bucket). Nối đường mũi tên từ Fargate-backend tới bucket này. Kèm chú thích: Storage Class: S3 Standard-IA (nếu ít truy cập) hoặc S3 Standard.

---------------------------------------

2. Thiếu thông tin ổ cứng EBS (Bắt buộc phải chỉ đích danh loại Volume)

* Vấn đề: Đề bài yêu cầu "Attach an EBS volume to your DB-tier... and note the volume type". Nhóm bạn dùng RDS (Managed Service) là rất xịn, nhưng nguyên lý dưới đáy RDS vẫn là ổ EBS. Giám khảo sẽ bắt bẻ nếu không nhắc đến Storage Type ở DB.
Cách vẽ thêm: Ngay bên dưới cụm icon RDS (Master & Standby Database), vẽ thêm icon ô vuông của Amazon EBS hoặc ghi hẳn Text box: Storage: EBS gp3. Kèm 1 dòng lý do ngắn: Reason: gp3 configures IOPS independently of storage capacity, cost-effective baseline.

---------------------------------------

3. Thiếu Định danh - IAM Roles (Lỗi sẽ bị trừ điểm nặng nhất)

* Vấn đề: Chưa thấy các thiết lập cấp quyền cho Fargate backend gọi các dịch vụ khác (ví dụ gọi S3, hoặc đẩy log lên CloudWatch).
Cách vẽ thêm: Gắn icon cái khiên có chìa khóa AWS IAM Role ngay cạnh cục ECS Cluster (Fargate-backend). Chú thích nhỏ: ECS Task Role: Allows backend to R/W S3 & send logs to CloudWatch. No hard-coded keys.

---------------------------------------

4. Thiếu ranh giới Security Groups (Tường lửa)

* Vấn đề: Bạn có WAF, có Route Table nhưng chưa thể hiện Security Group (SG).
Cách vẽ thêm: Hãy vẽ các đường viền nét đứt (Box đứt đoạn, có hình cái ổ khóa) bao quanh từng cụm để thể hiện Security Group:
ALB-SG: Chỉ cho phép Inbound Port 80/443 từ Internet/CloudFront.
App-SG (bao quanh Fargate): Chỉ cho phép Inbound Port của App (vd 8080) TỪ ALB-SG.
DB-SG (bao quanh RDS): Chỉ cho phép Inbound Port (3306 mặc định của MySQL hoặc 5432 của PostgreSQL) TỪ App-SG.
(Cái này rất tuyệt để chứng minh bạn hiểu rõ sự "kín kẽ" của mô hình 3-tier).

---------------------------------------

5. (Điểm cộng) Thêm mã hóa KMS

Vẽ thêm 1 icon AWS KMS ở góc sơ đồ, kéo mũi tên chỉ vào S3 và RDS để thể hiện: Mọi dữ liệu (Data at rest) đều được mã hóa bằng CMK (Customer Managed Key) từ KMS.
