# VPC + Networking Evidence

Trong phần này, nhóm sẽ trình bày về cấu trúc mạng đa tầng (Multi-Tier) và cách thức cấu hình bảo mật chuẩn giữa các tầng, cũng như phương thức tối ưu đường truyền dữ liệu với S3 Gateway Endpoint.

### 1. S3 Gateway Endpoint trong Route Table
![S3 Gateway Endpoint](Screenshot/S3%20Gateway%20Endpoint.png)

**Notes:** 
Nhóm đã thêm một S3 Gateway Endpoint vào route table của private subnet (định tuyến lưu lượng mạng cho S3 qua Prefix List `pl-68a54001` trực tiếp đến Target là `vpce-05a1f240aa0423a12`). Nhóm chọn dùng S3 Gateway Endpoint thay vì định tuyến qua NAT Gateway vì giải pháp này giữ mọi dữ liệu giao tiếp với S3 nằm hoàn toàn trong mạng nội bộ của AWS. Điều này không chỉ tăng cường bảo mật (vì không đi ra Internet public) mà còn loại bỏ hoàn toàn chi phí xử lý dữ liệu (data processing charges) của NAT Gateway.

---

### 2. Database Security Group Inbound Rule
![Database Inbound rules](Screenshot/Database%20Inbound%20rules.png)

**Notes:** 
Đối với Security Group của tầng Database, nhóm cấu hình Inbound Rules chỉ chấp nhận kết nối tại cổng của database với Nguồn (Source) là các Security Groups của tầng Application (ví dụ: ECS và Lambda SGs). Nhóm ưu tiên việc tham chiếu ID Security Group (Security Group referencing) thay vì dùng dải IP (subnet CIDRs) vì điều này tuân thủ tuyệt đối nguyên tắc quyền hạn tối thiểu (least privilege). Nó cũng tự động mở rộng (scales automatically): bất kỳ application instance mới nào được tạo ra và gắn App SG sẽ lập tức có quyền kết nối tới database mà không cần phải đi cập nhật dải IP theo cách thủ công.

---

### 3. Application Tier Security Groups (ECS & Lambda)
![ECS Inbound rules](Screenshot/ECS%20Inbound%20rules.png)

![Lambda Inbound rules](Screenshot/Lambda%20Inbound%20rules.png)

**Notes:** 
Tầng Application của nhóm gồm các tài nguyên tính toán như ECS và Lambda. Các Security Group của chúng đóng vai trò là lớp bảo mật trung gian. Ví dụ, ECS được cấu hình để chỉ nhận traffic đầu vào từ tầng Public (ALB Security Group). Điều này đảm bảo không có kết nối trực tiếp nào từ ngoài Internet có thể chọc vào các server ứng dụng của nhóm.

---

### 4. Public Tier Security Group (ALB)
![ALB Inbound rules](Screenshot/ALB%20Inbound%20rules.png)

**Notes:** 
Application Load Balancer (ALB) nằm ở tầng public. Security Group của nó được cấu hình để cho phép traffic HTTP/HTTPS vào từ ngoài Internet (`0.0.0.0/0`). Nó đóng vai trò là cổng vào public duy nhất để định tuyến các request đến với tầng private application của nhóm.

---

### Scenario dùng NACL thay cho Security Group

**Tình huống:** Nhóm sẽ sử dụng Network ACL (NACL) thay cho Security Group trong trường hợp cần phải deny traffic từ một địa chỉ IP độc hại cụ thể (ví dụ: đang bị tấn công DDoS từ một IP/dải IP đã biết).

**Lý giải:** Các Security Groups là stateful (có trạng thái) và chỉ hỗ trợ luật ALLOW (cho phép). Nếu port 443 đã mở cho mọi IP (`0.0.0.0/0`) trên Security Group, chúng ta không thể chặn riêng lẻ một IP xấu. Ngược lại, NACLs là stateless (không trạng thái), hoạt động ở cấp độ Subnet và hỗ trợ các luật DENY tường minh. Điều này làm cho NACLs trở thành công cụ thích hợp để chặn lưu lượng mạng độc hại ngay từ vòng ngoài (cấp Subnet) trước khi nó kịp chạm tới các instances của chúng ta.
