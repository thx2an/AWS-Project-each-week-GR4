## Q & A

🎯 Câu 1: Tại sao nhóm chọn ổ EBS loại gp3 cho hệ thống Database (RDS) của mình?

* Trả lời: "Nhóm em chọn gp3 vì đây là lựa chọn tối ưu nhất về hiệu năng/giá thành (Cost Performance) cho ổ đĩa đa dụng hiện nay. Khác với gp2 (tốc độ phụ thuộc vào dung lượng ổ), gp3 cung cấp sẵn baseline cố định là 3,000 IOPS và băng thông 125 MB/s ngay cả với dung lượng nhỏ nhất, mà giá thành lại rẻ hơn gp2 khoảng 20%. Nhóm em không chọn io2 vì hệ thống hiện tại chưa cần đến mức siêu độ trễ sub-millisecond với hàng chục ngàn IOPS của io2."

-------------------------------------------

️🎯 Câu 2: Fargate containers của nhóm bạn và Database nằm ở Private Subnet, vậy làm sao Fargate có thể có mạng Internet để gián tiếp tải các bản update hệ thống (hoặc gọi API bên thứ 3) được?
* Trả lời: "Nhóm em đã chuẩn bị sẵn cơ chế này trên sơ đồ bằng NAT Gateway. Fargate instance nằm trong Private Subnet lấy mạng ra ngoài thông qua Private Route Table (trỏ Route 0.0.0.0/0 về cái NAT Gateway). Bạn NAT Gateway này nằm ở Public Subnet và có một Public IP, nó sẽ thay mặt Fargate chuyển tiếp traffic ra Internet thông qua Internet Gateway (IGW), và chỉ cho phép kết nối Outbound, ngăn hoàn toàn mọi kết nối độc hại từ bên ngoài chiều Inbound chọc trực tiếp vào Fargate."

-------------------------------------------

️🎯 Câu 3: Làm thế nào để Backend Server (Fargate) kết nối với S3 bucket để tải/lưu file? Nếu ai đó yêu cầu bỏ Access Key vào thẳng file config của Fargate thì có được không?
* Trả lời: "Dạ dứt khoát KHÔNG dùng Access Key hard-code ạ. Đó là rủi ro rò rỉ dữ liệu (red flag) lớn nhất. Nhóm chúng em thiết lập kết nối bằng cách Gắn IAM Role (Task Role) cho ECS Fargate. Fargate sẽ tự động lấy thông tin xác thực tạm thời thông qua dịch vụ AWS STS để chứng minh quyền. Nhờ đó, ứng dụng kết nối lấy dữ liệu S3 một cách mượt mà và an toàn, dù có bị hack Server thì hacker cũng không lấy được Key vĩnh viễn để tấn công các dịch vụ khác."

-------------------------------------------

️🎯 Câu 4: Sự khác nhau cơ bản giữa việc dùng Security Group (SG) và NACL (Network ACL) mà nhóm bạn đang định tuyến là gì?
* Trả lời: "Security Group hoạt động ở lớp Instance/Resource. Quan trọng nhất nó là Stateful (Có trạng thái) - tức là nếu em mở Inbound cho một luồng vào, luồng Outbound phản hồi lại tự động được phép qua mà không cần cài rule trả về.
NACL hoạt động ở lớp bao quanh Subnet. Cơ chế là Stateless (Không trạng thái) - tức là nếu Request vào được, thì em phải setup thêm một rule Outbound mở các ephemeral ports để gói tin có thể đi ngược ra ngoài được."

-------------------------------------------

️🎯 Câu 5: Nếu 1 S3 Bucket của bạn đột nhiên "bị rò rỉ" trở thành Public access, bạn làm thế nào để phát hiện và phòng chống?
* Trả lời: "Thứ nhất về phòng chống, nhóm em sẽ bật tính năng Block Public Access ở mức Account/Bucket để chặn mọi nỗ lực pub bucket. Thứ 2, để theo dõi và phát hiện, nhóm dùng AWS Config hoặc cấu hình AWS GuardDuty quét cảnh báo nếu có sự xê dịch thay đổi bất thường về Bucket Policy, kết hợp với log ghi chép các hành động thay đổi môi trường từ hệ thống AWS CloudTrail, giúp tra cứu đích danh IAM User/Role nào vừa sửa Policy bucket thành public."
Security Group hoạt động ở lớp Instance/Resource. Quan trọng nhất nó là Stateful (Có trạng thái) - tức là nếu em mở Inbound cho một luồng vào, luồng Outbound phản hồi lại tự động được phép qua mà không cần cài rule trả về.
NACL hoạt động ở lớp bao quanh Subnet. Cơ chế là Stateless (Không trạng thái) - tức là nếu Request vào được, thì em phải setup thêm một rule Outbound mở các ephemeral ports để gói tin có thể đi ngược ra ngoài được.
