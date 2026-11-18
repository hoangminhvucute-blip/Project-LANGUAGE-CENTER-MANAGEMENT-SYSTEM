# 🌐 EPU LANGUAGE CENTER MANAGEMENT SYSTEM
**HỆ THỐNG QUẢN LÝ TRUNG TÂM NGOẠI NGỮ PHÂN TÁN (EPU LANG-CORE)**

![.NET]
![C#]
![TCP Socket]
![SQL Server]
![EPU]

---

### 📖 1. GIỚI THIỆU ĐỀ TÀI
* **Học phần:** Lập trình .NET
* **Khoa:** Công nghệ Thông tin - Trường Đại học Điện Lực (EPU)
* **Lớp:** D19CNPM3
* **Đề tài:** Xây dựng Hệ thống Quản lý Đào tạo Trung tâm Ngoại ngữ Đa cơ sở qua Giao thức Mạng Socket TCP.

Hệ thống mô phỏng quy trình vận hành thực tế của một trung tâm ngoại ngữ (Tiếng Anh, Tiếng Nhật, Tiếng Hàn, Tiếng Trung...) có nhiều quầy tư vấn và ghi danh cùng hoạt động đồng thời:
* **Máy trạm Quầy Ghi danh (Client Terminals):** Giao diện WinForms hiện đại dành cho nhân viên tư vấn tra cứu khóa học, đăng ký lớp cho học viên, thu học phí, xếp lịch học và in phiếu thu/chứng chỉ ngoại ngữ.
* **Máy chủ Trung tâm Ngoại ngữ (Lang-Core Server):** Tiếp nhận kết nối đồng thời từ nhiều quầy ghi danh qua giao thức TCP Socket đa luồng (Multithreaded Socket).
* **Cơ sở dữ liệu tập trung (Centralized SQL Database):** Quản lý học viên, giảng viên bản ngữ/Việt Nam, lớp học, phòng học, lịch giảng dạy, biên lai học phí và điểm thi 4 kỹ năng theo chuẩn toàn vẹn ACID.

---

### 👥 2. THÀNH VIÊN NHÓM THỰC HIỆN

| STT | Họ và Tên | Vai trò chính | Phân công nhiệm vụ cụ thể |
| :---: | :--- | :--- | :--- |
| **1** | **Hoàng Minh Vũ** *(Nhóm trưởng)* | **Backend Core Server & Database Architect** | • Thiết kế lược đồ CSDL chuẩn 3NF (`LangCenter_DB`) và khởi tạo dữ liệu mẫu.<br>• Lập trình máy chủ `Lang.Server` trên nền tảng `TcpListener` xử lý kết nối đa luồng (`Task.Run`).<br>• Xây dựng tầng nghiệp vụ `DatabaseHelper`, xử lý `SqlTransaction` và khóa tranh chấp sĩ số (`UPDLOCK`, `ROWLOCK`).<br>• Định nghĩa giao thức gói tin JSON (`Lang.Common`) và điều phối tiến độ GitHub. |
| **2** | **Phan Mạnh Trường** | **Frontend Client & Business Logic Developer** | • Thiết kế giao diện ứng dụng Quầy Ghi danh `Lang.Client` (WinForms) và khung điều hướng `MainForm`.<br>• Xây dựng bộ `UserControl` chức năng (Đăng nhập, Quản lý Học viên, Mở lớp & Xếp lịch, Đăng ký học, Nhập điểm, Thống kê).<br>• Lập trình lớp giao tiếp `LangClientSocket.cs` để gửi/nhận dữ liệu thời gian thực với Server.<br>• Tích hợp module Xuất phiếu thu học phí, in chứng chỉ ngoại ngữ và viết tài liệu báo cáo. |

---

### 🏗️ 3. KIẾN TRÚC HỆ THỐNG (SYSTEM ARCHITECTURE)
Hệ thống hoạt động theo mô hình **Client - Server Phân tán (Distributed Client-Server Model)**:

```text
+-----------------------+       TCP Socket (JSON)       +-----------------------+       ADO.NET / SQL       +-----------------------+

|   Quầy Ghi Danh 01    | <===========================> |                       | <=======================> |                       |
|    (Lang.Client)      |         Port: 9999            |   Lang-Core Server    |     SqlTransaction        |    SQL Server 2022    |
+-----------------------+                               |     (Lang.Server)     |    (UPDLOCK, ROWLOCK)     |    (LangCenter_DB)    |

                                                        |                       |                           |                       |
+-----------------------+       TCP Socket (JSON)       |  Multithreaded Pool   |                           |  ACID Centralized DB  |

|   Quầy Ghi Danh 02    | <===========================> |      (Task.Run)       |                           |                       |
|    (Lang.Client)      |         Port: 9999            +-----------------------+                           +-----------------------+
+-----------------------+
```

---

### ⚡ 4. TRỌNG TÂM KỸ THUẬT & ĐIỂM SÁNG ĐỒ ÁN

#### 🎯 4.1. Bài toán Đa luồng (Multithreading) & Tranh chấp Sĩ số lớp (Concurrency Lock)
* **Vấn đề thực tiễn:** Lớp Tiếng Anh giao tiếp IELTS K19 có sĩ số tối đa 15 học viên và hiện đã có 14 người đăng ký (chỉ còn đúng 1 slot cuối). Nếu 2 nhân viên ở 2 quầy tư vấn khác nhau cùng bấm xác nhận đăng ký cho 2 học viên vào cùng 1 mili-giây, hệ thống có bị nhận lố thành 16/15 học viên hay không?
* **Giải pháp kỹ thuật:**
  * **Đa luồng Socket:** Mỗi máy trạm quầy ghi danh kết nối vào Server được đẩy vào một luồng độc lập qua `Task.Run(() => HandleClient(client))` trong ThreadPool của .NET, bảo đảm phục vụ hàng chục quầy cùng lúc mà không bị treo Server.
  * **Khóa cấp độ dòng CSDL (Pessimistic Row-Level Lock):**
    ```sql
    SELECT SiSoHienTai, SiSoToiDa, TrangThai FROM LopHoc WITH (UPDLOCK, ROWLOCK) WHERE MaLop = @maLop
    ```
  * **Cơ chế hoạt động:**
    * `ROWLOCK`: Chỉ khóa duy nhất bản ghi của lớp ngoại ngữ đang đăng ký, các lớp khác vẫn giao dịch song song 100% với độ trễ 0ms.
    * `UPDLOCK`: Giữ cờ khóa cập nhật ngay từ lúc kiểm tra sĩ số, buộc quầy thứ 2 phải chờ quầy thứ 1 `COMMIT` hoặc `ROLLBACK` xong mới được đọc tiếp (lúc này sĩ số đã là 15/15 và tự động báo lớp đã đầy).

#### 🛡️ 4.2. Kiểm tra Trùng lịch & An toàn Dữ liệu (Security)
* **Thuật toán kiểm tra trùng lịch tự động:** Khi xếp lớp hoặc đăng ký khóa học mới, hệ thống tự động đối chiếu Ca học và Thứ trong tuần của Học viên, Giảng viên và Phòng học để ngăn chặn tuyệt đối việc xếp trùng lịch.
* **Bảo mật & Phân quyền:** Mã hóa mật khẩu nhân viên bằng SHA-256, phân quyền rõ ràng giữa **Admin (Quản lý trung tâm)**, **Nhân viên tư vấn ghi danh** và **Giảng viên nhập điểm**.

---

### 💻 5. CÁC TÍNH NĂNG CHÍNH CỦA HỆ THỐNG

| Tính năng | Mô tả chi tiết |
| :--- | :--- |
| **1. Đăng nhập & Phân quyền** | Đăng nhập bằng tài khoản nhân viên. Tự động ẩn/hiện menu chức năng theo vai trò (Quản lý, Tư vấn viên/Thu ngân, Giảng viên). |
| **2. Quản lý Học viên & Giảng viên** | Thêm, sửa, xóa, tìm kiếm thông tin học viên, giảng viên (phân loại giảng viên Việt Nam / Bản ngữ, chuyên môn ngôn ngữ). Tự động sinh mã học viên (`HV2026xxxx`). |
| **3. Mở lớp & Xếp lịch học** | Tạo lớp theo khóa ngoại ngữ (IELTS, TOEIC, Tiếng Nhật N5-N3, Tiếng Hàn TOPIK, Tiếng Trung HSK), gán phòng học và giảng viên có kiểm tra trùng lịch. |
| **4. Đăng ký Lớp & Thu học phí** | Ghi danh học viên vào lớp (kiểm tra sĩ số thời gian thực), áp dụng mã giảm giá/học bổng, cập nhật trạng thái đã đóng học phí. |
| **5. Quản lý Điểm & Cấp chứng chỉ** | Nhập điểm giữa kỳ, cuối kỳ (hoặc điểm 4 kỹ năng Nghe - Nói - Đọc - Viết); tự động xét điều kiện đạt và xuất chứng chỉ hoàn thành khóa học. |
| **6. In Biên lai & Báo cáo Doanh thu** | Xuất phiếu thu học phí ra file bản cứng (`BienLai_HocPhi_*.txt` hoặc PDF) và vẽ biểu đồ thống kê doanh thu theo từng ngoại ngữ. |

---

### 🗂️ 6. CẤU TRÚC THƯ MỤC DỰ ÁN

```text
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
├── .gitattributes                    # Chuẩn hóa ngắt dòng Git
└── README.md                         # Tài liệu hướng dẫn chính của dự án
```

---

### 🚀 7. HƯỚNG DẪN CÀI ĐẶT & CHẠY THỬ NGHIỆM (QUICK START)

#### 📌 Yêu cầu môi trường:
* **Hệ điều hành:** Windows 10 / Windows 11
* **IDE:** Visual Studio 2022 (đã cài gói *.NET desktop development*)
* **SDK:** .NET 10.0 (hoặc .NET 8.0 trở lên)
* **CSDL:** Microsoft SQL Server 2022 (bản Developer hoặc SQL Express) + SSMS

#### ⚙️ Các bước thực hiện:

**Bước 1: Khởi tạo Cơ sở Dữ liệu**
1. Mở phần mềm SQL Server Management Studio (SSMS).
2. Mở file `Database/LangCenter_DB.sql`.
3. Nhấn **Execute (`F5`)** để tạo CSDL `LangCenter_DB` và nạp toàn bộ dữ liệu mẫu.

**Bước 2: Khởi động Lang-Core Server**
1. Mở Solution `EPU_LangCenterSystem.slnx` bằng Visual Studio 2022.
2. Chuột phải vào project `Lang.Server` → chọn **Set as Startup Project**.
3. Nhấn `F5` (hoặc `Ctrl + F5`) để chạy Server. Cửa sổ Console màu đen sẽ hiện thông báo:
