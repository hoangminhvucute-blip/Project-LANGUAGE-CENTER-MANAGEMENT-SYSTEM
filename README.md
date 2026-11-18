
🌐 EPU LANGUAGE CENTER MANAGEMENT SYSTEM

HỆ THỐNG QUẢN LÝ TRUNG TÂM NGOẠI NGỮ PHÂN TÁN (EPU LANG-CORE)
.NET 10 C# TCP Socket SQL Server 2022 WinForms EPU D19CNPM3

📖 1. GIỚI THIỆU ĐỀ TÀI

• Học phần: Lập trình .NET
• Khoa: Công nghệ Thông tin - Trường Đại học Điện Lực (EPU)
• Lớp: D19CNPM3
• Đề tài: Xây dựng Hệ thống Quản lý Đào tạo Trung tâm Ngoại ngữ Đa cơ sở qua Giao thức Mạng Socket TCP.
Hệ thống mô phỏng quy trình vận hành thực tế của một trung tâm ngoại ngữ (Tiếng Anh, Tiếng Nhật, Tiếng Hàn, Tiếng Trung...) có nhiều quầy tư vấn và ghi danh cùng hoạt động đồng thời:
• Máy trạm Quầy Ghi danh (Client Terminals): Giao diện WinForms hiện đại dành cho nhân viên tư vấn tra cứu khóa học, đăng ký lớp cho học viên, thu học phí, xếp lịch học và in phiếu thu/chứng chỉ ngoại ngữ.
• Máy chủ Trung tâm Ngoại ngữ (Lang-Core Server): Tiếp nhận kết nối đồng thời từ nhiều quầy ghi danh qua giao thức TCP Socket đa luồng (Multithreaded Socket).
• Cơ sở dữ liệu tập trung (Centralized SQL Database): Quản lý học viên, giảng viên bản ngữ/Việt Nam, lớp học, phòng học, lịch giảng dạy, biên lai học phí và điểm thi 4 kỹ năng theo chuẩn toàn vẹn ACID.

👥 2. THÀNH VIÊN NHÓM THỰC HIỆN (2 THÀNH VIÊN)


STT	Họ và Tên	Vai trò chính	Phân công nhiệm vụ cụ thể
1	Hoàng Minh Vũ (Nhóm trưởng)	Backend Core Server & Database Architect	
• Thiết kế lược đồ CSDL chuẩn 3NF (LangCenter_DB) và khởi tạo dữ liệu mẫu.
• Lập trình máy chủ Lang.Server trên nền tảng TcpListener xử lý kết nối đa luồng (Task.Run).
• Xây dựng tầng nghiệp vụ DatabaseHelper, xử lý SqlTransaction và khóa tranh chấp sĩ số (UPDLOCK, ROWLOCK).
• Định nghĩa giao thức gói tin JSON (Lang.Common) và điều phối tiến độ GitHub.
2	Phan Mạnh Trường	Frontend Client & Business Logic Developer	
• Thiết kế giao diện ứng dụng Quầy Ghi danh Lang.Client (WinForms) và khung điều hướng MainForm.
• Xây dựng bộ UserControl chức năng (Đăng nhập, Quản lý Học viên, Mở lớp & Xếp lịch, Đăng ký học, Nhập điểm, Thống kê).
• Lập trình lớp giao tiếp LangClientSocket.cs để gửi/nhận dữ liệu thời gian thực với Server.
• Tích hợp module Xuất phiếu thu học phí, in chứng chỉ ngoại ngữ và viết tài liệu báo cáo.


⚡ 3. TRỌNG TÂM KỸ THUẬT & ĐIỂM SÁNG ĐỒ ÁN


🎯 3.1. Bài toán Đa luồng (Multithreading) & Tranh chấp Sĩ số lớp (Concurrency Lock)

• Vấn đề thực tiễn: Lớp Tiếng Anh giao tiếp IELTS K19 có sĩ số tối đa 15 học viên và hiện đã có 14 người đăng ký (chỉ còn đúng 1 slot cuối). Nếu 2 nhân viên ở 2 quầy tư vấn khác nhau cùng bấm xác nhận đăng ký cho 2 học viên vào cùng 1 mili-giây, hệ thống có bị nhận lố thành 16/15 học viên hay không?
• Giải pháp kỹ thuật:
	• Đa luồng Socket: Mỗi máy trạm quầy ghi danh kết nối vào Server được đẩy vào một luồng độc lập qua Task.Run(() => HandleClient(client)) trong ThreadPool của .NET.
	• Khóa cấp độ dòng CSDL (Pessimistic Row-Level Lock):sql
SELECT SiSoHienTai, SiSoToiDa, TrangThai FROM LopHoc WITH (UPDLOCK, ROWLOCK) WHERE MaLop = @maLop
Hãy thận trọng khi sử dụng mã.
	• Cơ chế hoạt động: ROWLOCK chỉ khóa duy nhất bản ghi của lớp ngoại ngữ đang đăng ký, giúp các lớp khác vẫn đăng ký song song 100% không độ trễ. UPDLOCK giữ cờ khóa cập nhật ngay lúc kiểm tra sĩ số, buộc quầy thứ 2 phải chờ quầy thứ 1 COMMIT xong mới được đọc tiếp (lúc này sĩ số đã là 15/15 và tự động báo lớp đã đầy).

🛡️ 3.2. Kiểm tra Trùng lịch & An toàn Dữ liệu

• Thuật toán kiểm tra trùng lịch tự động: Khi xếp lớp hoặc đăng ký khóa học mới, hệ thống tự động đối chiếu Ca học và Thứ trong tuần của Học viên, Giảng viên và Phòng học để ngăn chặn tuyệt đối việc xếp trùng lịch.
• Bảo mật & Phân quyền: Mã hóa mật khẩu nhân viên bằng SHA-256, phân quyền rõ ràng giữa Admin (Quản lý trung tâm), Nhân viên tư vấn ghi danh và Giảng viên nhập điểm.

💻 4. CÁC TÍNH NĂNG CHÍNH CỦA HỆ THỐNG


Tính năng	Mô tả chi tiết
1. Đăng nhập & Phân quyền	Đăng nhập bằng tài khoản nhân viên. Tự động ẩn/hiện menu chức năng theo vai trò (Quản lý, Tư vấn viên/Thu ngân, Giảng viên).
2. Quản lý Học viên & Giảng viên	Thêm, sửa, xóa, tìm kiếm thông tin học viên, giảng viên (phân loại giảng viên Việt Nam / Bản ngữ, chuyên môn ngôn ngữ). Tự động sinh mã học viên (HV2026xxxx).
3. Mở lớp & Xếp lịch học	Tạo lớp theo khóa ngoại ngữ (IELTS, TOEIC, Tiếng Nhật N5-N3, Tiếng Hàn TOPIK, Tiếng Trung HSK), gán phòng học và giảng viên có kiểm tra trùng lịch.
4. Đăng ký Lớp & Thu học phí	Ghi danh học viên vào lớp (kiểm tra sĩ số thời gian thực), áp dụng mã giảm giá/học bổng, cập nhật trạng thái đã đóng học phí.
5. Quản lý Điểm & Cấp chứng chỉ	Nhập điểm giữa kỳ, cuối kỳ (hoặc điểm 4 kỹ năng Nghe - Nói - Đọc - Viết); tự động xét điều kiện đạt và xuất chứng chỉ hoàn thành khóa học.
6. In Biên lai & Báo cáo Doanh thu	Xuất phiếu thu học phí ra file bản cứng (BienLai_HocPhi_*.txt hoặc PDF) và vẽ biểu đồ thống kê doanh thu theo từng ngoại ngữ.


🗂️ 5. CẤU TRÚC THƯ MỤC DỰ ÁN (SOLUTION ARCHITECTURE)

text
EPU_LangCenterSystem/
│
├── Lang.Client/                      # [FRONTEND] Ứng dụng Quầy Ghi danh & Quản lý (WinForms)
│   ├── Screens/                      # Bộ các màn hình UserControl chuyển đổi mượt mà
│   │   ├── UcLogin.cs                # Màn hình Đăng nhập hệ thống
│   │   ├── UcDashboard.cs            # Màn hình Tổng quan & Biểu đồ doanh thu
│   │   ├── UcStudents.cs             # Màn hình Quản lý hồ sơ Học viên
│   │   ├── UcClasses.cs              # Màn hình Quản lý Khóa ngoại ngữ & Xếp lịch lớp
│   │   ├── UcEnrollment.cs           # Màn hình Đăng ký lớp & Thu học phí (Xử lý tranh chấp)
│   │   └── UcGrades.cs               # Màn hình Nhập điểm & Xét cấp chứng chỉ
│   ├── LangClientSocket.cs           # Lớp chuyên trách gửi/nhận gói tin JSON qua Socket TCP
│   ├── MainForm.cs                   # Khung giao diện chính (Sidebar Menu + Vùng hiển thị)
│   └── FormInvoice.cs                # Hộp thoại xem trước và in phiếu thu học phí
│
├── Lang.Server/                      # [BACKEND] Máy chủ Trung tâm Ngoại ngữ (Console App)
│   └── Program.cs                    # TcpListener lắng nghe Port 9999, Đa luồng Task.Run
│
├── Lang.Common/                      # [SHARED CORE] Thư viện nghiệp vụ & Giao thức chung
│   ├── DatabaseHelper.cs             # Kết nối SQL Server, SqlTransaction, Row-level Lock
│   ├── RequestPacket.cs              # Định nghĩa cấu trúc gói tin Client gửi lên Server
│   └── ResponsePacket.cs             # Định nghĩa cấu trúc gói tin Server trả về Client
│
├── Database/                         # [DATABASE] Kịch bản Cơ sở dữ liệu
│   └── LangCenter_DB.sql             # File script tạo các bảng quan hệ, khóa ngoại và dữ liệu mẫu
│
├── Docs/                             # [DOCUMENTS] Tài liệu đồ án, báo cáo chuẩn EPU
├── EPU_LangCenterSystem.slnx         # File Solution chính (.NET 10 / Visual Studio 2022)
├── .gitignore                        # Cấu hình bỏ qua file tạm build (.vs, bin, obj)
└── README.md                         # Tài liệu hướng dẫn chính của dự án
Hãy thận trọng khi sử dụng mã.

🚀 6. HƯỚNG DẪN CÀI ĐẶT & CHẠY THỬ NGHIỆM (QUICK START)


📌 Yêu cầu môi trường:

• Hệ điều hành: Windows 10 / Windows 11
• IDE: Visual Studio 2022 (đã cài gói .NET desktop development)
• SDK: .NET 10.0 (hoặc .NET 8.0 trở lên)
• CSDL: Microsoft SQL Server 2022 (bản Developer hoặc SQL Express) + SSMS

⚙️ Các bước thực hiện:

1. Khởi tạo Cơ sở Dữ liệu:
	• Mở SQL Server Management Studio (SSMS), chạy file Database/LangCenter_DB.sql (F5) để tạo CSDL LangCenter_DB và nạp dữ liệu mẫu.
2. Khởi động Lang-Core Server:
	• Mở Solution EPU_LangCenterSystem.slnx bằng Visual Studio 2022.
	• Chuột phải vào project Lang.Server → chọn Set as Startup Project và nhấn F5 để chạy Server lắng nghe trên cổng 9999.
3. Khởi động Máy trạm Quầy Ghi danh (Client):
	• Chuột phải vào project Lang.Client → chọn Debug → Start New Instance (hoặc mở trực tiếp file Lang.Client.exe).
	• Mở cùng lúc 2 cửa sổ Lang.Client.exe để mô phỏng 2 quầy tư vấn ghi danh hoạt động song song và kiểm thử tính năng khóa tranh chấp suất đăng ký cuối cùng của lớp ngoại ngữ.
