<img width="1848" height="893" alt="image" src="https://github.com/user-attachments/assets/f43ea957-918c-4892-baa9-307e8d7ee549" />TÀI LIỆU TỔNG THỂ TRIỂN KHAI HẠ TẦNG IT DOANH NGHIỆP
----

# 1. Mục tiêu của toàn bộ hệ thống

Mục tiêu không chỉ là:

> **"Cài Domain Controller và Join Domain."**

Mà là xây dựng một hệ thống trong đó:
```
                 INTERNET
                    │
                   VNPT
                    │
                pfSense
          Firewall / Routing / NAT
                    │
             Cisco Catalyst
          VLAN / 802.1X / RADIUS
                    │
       ┌────────────┼────────────┐
       │            │            │
     Client       Server       WiFi
       │            │            │
       └────────────┼────────────┘
                    │
                 AD DS
                    │
      ┌─────────────┼─────────────┐
      │             │             │
     DNS           DHCP          GPO
      │             │             │
      └─────────────┼─────────────┘
                    │
                  NPS
                    │
                 AD CS
                    │
                Certificate
                    │
                 EAP-TLS
                    │
               Dynamic VLAN
```
Sau cùng hệ thống đạt được:
- Người dùng được quản lý tập trung bằng AD.
- Máy tính phải thuộc Domain mới được xác thực mạng.
- Certificate được cấp tự động.
- 802.1X xác thực máy.
- NPS quyết định máy được truy cập hay không.
- Cisco đưa thiết bị vào VLAN tương ứng.
- pfSense kiểm soát lưu lượng giữa các VLAN và Internet.
- File Server phân quyền theo AD.
- GPO quản lý máy trạm.
- Backup bảo vệ AD/DNS/DHCP/GPO/PKI.
- Monitoring/Audit cho phép biết hệ thống đang hoạt động thế nào và ai đã thực hiện thay đổi gì.

# 2. Kiến trúc tổng thể

Về mặt chức năng, nên chia thành 5 lớp:
```
LAYER 1
NETWORK
VNPT
pfSense
Cisco
VLAN
        ↓
LAYER 2
IDENTITY
AD DS
Users
Groups
OU
        ↓
LAYER 3
SECURITY
AD CS
Certificate
NPS
802.1X
Dynamic VLAN
        ↓
LAYER 4
SERVICES
DNS
DHCP
GPO
File Server
        ↓
LAYER 5
OPERATIONS
Backup
Monitoring
Audit
Documentation
```
Đây là cách nên trình bày trong tài liệu doanh nghiệp vì mỗi thành phần có một **vai trò riêng**, tránh việc một server làm tất cả mọi thứ mà không có phân tách.

# 3. Quy hoạch địa chỉ IP

Đây là một trong những phần cần đặc biệt chú ý.

Kiến trúc cuối cùng chúng ta đã hướng tới là:

| VLAN | Chức năng         | Network           | Gateway          |
| ---: | ----------------- | ----------------- | ---------------- |
|   10 | User              | `192.168.10.0/24` | `192.168.10.1`   |
|   20 | Accounting        | `192.168.20.0/24` | `192.168.20.1`   |
|   30 | Sales             | `192.168.30.0/24` | `192.168.30.1`   |
|   40 | WiFi              | `192.168.40.0/24` | `192.168.40.1`   |
|   50 | IT                | `192.168.50.0/24` | `192.168.50.1`   |
|   90 | Server/Management | `192.168.90.0/24` | `192.168.90.254` |

Server:
```
DC01
192.168.90.10
```
File Server:
```
FS01
192.168.90.20
```
pfSense:
```
VLAN 90
192.168.90.254
```
**Lưu ý quan trọng**

Không nên để:
```
VLAN 10
VLAN 20
VLAN 30
VLAN 40
VLAN 50
```
tất cả cùng nằm trong `192.168.90.0/24.`

Nếu đã gọi là VLAN phân tách phòng ban thì nên có **subnet riêng.**

# 4. Chương 1 — Xây dựng Network Foundation

Mục tiêu của chương này là tạo nền móng.

Mô hình:
```
VNPT
 │
 ▼
pfSense
 │
 ▼
Cisco Catalyst 2960S
 │
 ├── VLAN 10
 ├── VLAN 20
 ├── VLAN 30
 ├── VLAN 40
 ├── VLAN 50
 └── VLAN 90
```

**Thành phần**

**pfSense**
- WAN
- NAT
- Firewall
- Gateway
- Inter-VLAN Routing

**Cisco**
- Access Switch
- VLAN
- Trunk
- Access Port
- 802.1X

**Điểm cần lưu ý**

Không nên đưa 802.1X vào ngay.

Thứ tự phải là:
```
VLAN
 ↓
Routing
 ↓
DHCP
 ↓
DNS
 ↓
Domain
 ↓
802.1X
```
Nếu VLAN chưa chạy mà bật 802.1X thì việc troubleshooting sẽ rất khó.

# 5. Chương 2 — pfSense Foundation

pfSense là biên giới mạng, không phải Domain Controller.

Nó chịu trách nhiệm:
```
WAN
NAT
Firewall
Routing
VLAN Gateway
VPN
Logging
```
Không nên yêu cầu pfSense tự biết:

> **Máy này đã Join Domain chưa?**

Việc xác thực Domain nên giao cho:
```
AD + Certificate + NPS + 802.1X
```
Sau đó pfSense chỉ thực hiện chính sách dựa trên VLAN/IP.

# 6. Chương 3 — Cisco VLAN và Switching

Cisco chịu trách nhiệm:
```
VLAN
Access Port
Trunk
STP
802.1X
RADIUS
Dynamic VLAN
```
Cần phân biệt:

**Access**

PC:
```
PC
 │
Gi0/10
 │
VLAN 10
```

**Trunk**

pfSense ↔ Cisco:
```
pfSense
   │
   │ 802.1Q
   │ VLAN 10,20,30,40,50,90
   ▼
Cisco
```

**Lưu ý**

Không cấu hình tất cả port thành trunk.

Thông thường:
```
pfSense ↔ Cisco = Trunk
Cisco ↔ PC = Access
```
Sau này 802.1X sẽ được áp dụng trên access port.

# 7. Chương 4 — AD DS

Domain Controller là trung tâm Identity.

Ví dụ:
```
DC01
192.168.90.10
company.local
```
AD quản lý:
```
Users
Computers
Groups
OU
Authentication
Authorization
GPO
```

**Nguyên tắc quan trọng**

DC phải sử dụng **IP tĩnh.**

Không nên:
```
DC01 → DHCP
```
mà:
```
DC01
192.168.90.10
```

## 8. Chương 5 — DNS

DNS là thành phần **cực kỳ quan trọng** đối với AD.

Domain:
```
company.local
```
DNS:
```
192.168.90.10
```
Domain PC:
```
DNS = 192.168.90.10
```
Không nên cấu hình Domain PC:
```
DNS = 8.8.8.8
DNS = 1.1.1.1
```
thay cho DNS nội bộ.

Mô hình đúng:
```
PC
 │
 ▼
192.168.90.10
Windows DNS
 │
 ▼
Forwarder
 │
 ▼
Internet DNS
```

**Đây cũng là nguyên nhân của rất nhiều lỗi Join Domain.**

Nếu:
```
ping DC
OK
```
nhưng:
```
Join Domain
FAIL
```
thì phải kiểm tra DNS trước.

# 9. Chương 6 — Active Directory Structure

Thiết kế OU trước khi tạo hàng loạt User.

Ví dụ:
```
company.local
│
├── OU=Users
│   ├── IT
│   ├── Accounting
│   ├── Sales
│   └── HR
│
├── OU=Computers
│   ├── IT
│   ├── Accounting
│   ├── Sales
│   └── General
│
├── OU=Servers
│
├── OU=Groups
│
└── OU=Service Accounts
```

**Lưu ý**

OU dùng để:
- áp GPO
- tổ chức đối tượng
- ủy quyền quản trị

Security Group dùng để:
- cấp quyền

Không nên nhầm hai khái niệm này.

## 10. Chương 7 — DHCP

DHCP có nhiệm vụ cấp:
```
IP
Subnet Mask
Gateway
DNS
DNS Suffix
```
Ví dụ VLAN 10:
```
192.168.10.50
-
192.168.10.200
```
Gateway:
```
192.168.10.1
````
DNS:
```
192.168.90.10
```
Domain:
```
company.local
```

**Điểm cần lưu ý**

DHCP Server ở VLAN 90 nhưng Client ở VLAN 10/20/30...

Do DHCP broadcast không đi qua router nên cần:
```
DHCP Relay
```
trên pfSense.

Mô hình:
```
Client VLAN10
      ↓
pfSense DHCP Relay
      ↓
192.168.90.10
      ↓
Windows DHCP
```

# 11. Chương 8 — OU / User / Group / GPO

Đây là tầng Identity Management.

Nguyên tắc:
```
User
 ↓
Group
 ↓
Permission
```
Không nên:
```
User
 ↓
Cấp quyền trực tiếp
```
Ví dụ:
```
user01
 ↓
GG_ACCOUNTING
 ↓
Resource
```

# 12. Chương 9 — Group Policy

GPO dùng để quản lý tập trung:
```
Password
Windows Update
Firewall
Screen Lock
USB
Security
Certificate
802.1X
Drive Mapping
```

**Điểm cần lưu ý**

Không tạo một GPO khổng lồ chứa tất cả mọi thứ.

Nên chia:
```
GPO - Security
GPO - Windows Update
GPO - Certificate Auto Enrollment
GPO - 802.1X
GPO - Drive Mapping
GPO - Workstation
```
Dễ kiểm tra và rollback hơn.

# 13. Chương 10 — Security Baseline

Mục tiêu là chuẩn hóa Windows Client/Server.

Ví dụ:
```
Password Policy
Account Lockout
Windows Firewall
Screen Lock
USB
Defender
Windows Update
Audit
```

**Điểm cần lưu ý**

Không áp chính sách quá mạnh ngay lập tức.

Ví dụ:
```
Disable USB
Block all PowerShell
Block all local admin
```
nếu chưa test có thể làm IT không thể vận hành máy.

Phải:
```
Test OU
 ↓
Pilot
 ↓
Production
```

# 14. Chương 11 — Domain Security / Administration

Đây là lớp quản trị.

Nguyên tắc:

> **Least Privilege.**

Không sử dụng Domain Admin cho công việc hàng ngày.

Nên có:
```
User Account
+
Admin Account
```
Ví dụ:
```
toan
adm-toan
```
Tài khoản admin chỉ dùng khi cần quản trị.

# 15. Chương 12 — AD CS / Enterprise PKI

AD CS cung cấp Certificate.

Mô hình:
```
Enterprise CA
      │
      ├── Computer Certificate
      │
      └── NPS Certificate
```
Certificate dùng cho:
```
EAP-TLS
802.1X
Computer Authentication
```

**Điểm đặc biệt quan trọng**

Private Key của CA phải được bảo vệ.

Mất CA:
```
CA Private Key
Certificate
802.1X
```
có thể ảnh hưởng toàn bộ hệ thống xác thực.

Do đó Chương 19 phải backup CA.

# 16. Chương 13 — EAP-TLS + NPS

NPS là RADIUS Server.

Luồng:
```
PC
 ↓
802.1X
 ↓
Cisco
 ↓
RADIUS
 ↓
NPS
 ↓
AD/Certificate
```
EAP-TLS dùng Certificate thay vì chỉ username/password.

Đây là thành phần giúp đạt mục tiêu:

> **Máy không thuộc hệ thống Domain/không có certificate hợp lệ thì không được xác thực mạng.**

**Điểm cần lưu ý**

Không bật toàn bộ 802.1X ngay.

Phải:
```
1 PC
 ↓
1 Cisco port
 ↓
NPS
 ↓
Certificate
 ↓
Authentication
 ↓
Test
```
sau đó mới rollout.

# 17. Chương 14 — Dynamic VLAN

Đây là phần kết hợp:
```
AD
+
NPS
+
Cisco
```

Ví dụ:
```
GG_IT
 ↓
NPS
 ↓
VLAN 50
```

```
GG_ACCOUNTING
 ↓
NPS
 ↓
VLAN 20
```

```
GG_SALES
 ↓
NPS
 ↓
VLAN 30
```
Cisco nhận:
```
Tunnel-Type
Tunnel-Medium-Type
Tunnel-Pvt-Group-ID
```
và đưa client vào VLAN.

**Điểm quan trọng**

**Dynamic VLAN không phải chức năng của pfSense.**

Luồng đúng:
```
AD/NPS
 ↓
RADIUS
 ↓
Cisco
 ↓
Dynamic VLAN
 ↓
pfSense
```

# 18. Chương 15 — pfSense Firewall giữa các VLAN

Sau Dynamic VLAN, pfSense trở thành lớp kiểm soát giữa các VLAN.

Ví dụ:
```
VLAN10
 ↓
pfSense
 ↓
Internet
```
nhưng:
```
VLAN10
 ↓
VLAN20
```
có thể bị Block.

Mục tiêu:
```
USER → Internet       ALLOW
USER → DNS/DC         ALLOW cần thiết
USER → Accounting     BLOCK
Sales → Accounting    BLOCK
WiFi → Server         BLOCK
IT → Server           ALLOW
```

**Lưu ý cực kỳ quan trọng**

Firewall rule của pfSense được xử lý theo thứ tự.

Các rule cụ thể phải đặt đúng vị trí.

Ví dụ:
```
BLOCK VLAN10 → VLAN20
ALLOW VLAN10 → ANY
```
phải đặt Block trước Allow.

## 19. Chương 16 — Chính sách Internet theo AD/NPS/VLAN

Đây là nơi hoàn thành mục tiêu ban đầu của bạn.

Không nên làm:
```
pfSense
 ↓
kiểm tra Join Domain
```
Thay vào đó:
```
Domain PC
 ↓
Certificate
 ↓
802.1X
 ↓
NPS
 ↓
Access-Accept
 ↓
Dynamic VLAN
 ↓
pfSense
 ↓
Internet
```
Máy không xác thực:
```
No Certificate
 ↓
NPS Reject
 ↓
Không được authorize
 ↓
Không Internet
```
Đây là kiến trúc tốt hơn việc chỉ dựa trên MAC/IP.

# 20. Chương 17 — DHCP/DNS/GPO nâng cao

Đây là giai đoạn tối ưu hóa.

Bao gồm:

**DHCP**
```
Scope từng VLAN
Exclusion
Reservation
DHCP Options
DHCP Relay
```

**DNS**
```
AD-integrated DNS
Forwarder
Reverse Lookup Zone
DNS Health
```

**GPO**
```
Auto Enrollment
802.1X
WiFi
Drive Mapping
Security
Update
Audit
```

**Nguyên tắc**
```
DHCP
 ↓
Gateway
 ↓
pfSense

DHCP
 ↓
DNS
 ↓
DC01

DNS
 ↓
AD
```

# 21. Chương 18 — File Server

File Server nên tách khỏi DC.
```
DC01
192.168.90.10

FS01
192.168.90.20
```

FS01:
```
Join Domain
 ↓
File Server Role
 ↓
SMB
 ↓
NTFS
```

# 22. Phân quyền File Server — AGDLP

Đây là nguyên tắc cần đưa vào tài liệu chính thức.
```
User
 ↓
Global Group
 ↓
Domain Local Group
 ↓
Permission
```
Ví dụ:
```
NguyenA
 ↓
GG_ACCOUNTING
 ↓
DL_FS_ACCOUNTING_RW
 ↓
\\FS01\Accounting
```
Không cấp quyền trực tiếp:
```
NguyenA → Folder
```
trừ trường hợp đặc biệt.

# 23. Share Permission + NTFS Permission

Phải hiểu rằng có **hai lớp quyền.**
```
SMB Share Permission
        +
NTFS Permission
        ↓
Effective Permission
```
Do đó phải thiết kế có chủ đích.

Ví dụ:
```
DL_FS_ACCOUNTING_RW
    Share = Change
    NTFS  = Modify
```

# 24. GPO Drive Mapping

Người dùng đăng nhập:
```
Domain Login
 ↓
GPO
 ↓
Security Group
 ↓
Drive Mapping
```

Ví dụ:
```
Accounting
 ↓
A:
\\FS01\Accounting
```

Sales:
```
S:
\\FS01\Sales
```
Đây là cách quản lý tập trung tốt hơn việc cấu hình thủ công từng máy.

# 25. Chương 19 — Backup

Backup phải bảo vệ tối thiểu:
```
AD
DNS
DHCP
GPO
AD CS
NPS
File Server
```
Đặc biệt:
```
AD System State
CA Private Key
CA Database
GPO
DHCP
NPS
```

# 26. Quy tắc 3-2-1

Mô hình:
```
             DC01
               │
        ┌──────┴──────┐
        │             │
       NAS        Backup Storage
        │
        └──────┬──────┘
               │
            Offsite
```
Không nên chỉ có:
```
DC01 → D:\Backup
```
vì nếu server chết thì backup cũng chết.

# 27. Điểm quan trọng nhất của Backup

Không chỉ:

> **Backup thành công.**

Mà phải:

> **Restore thành công.**

Quy trình:
```
Backup
 ↓
Verify
 ↓
Test Restore
 ↓
Document
```
Một backup chưa từng restore thử thì **chưa thể coi là phương án Disaster Recovery đáng tin cậy.**

# 28. Chương 20 — Monitoring

Hệ thống cần giám sát:
```
pfSense
Cisco
DC01
FS01
ESXi
NPS
AD CS
DHCP
DNS
```
Theo dõi:
```
CPU
RAM
Disk
Network
Service
Interface
Error
Availability
```

# 29. Audit

Audit cần trả lời được:

> Ai?

> Làm gì?

> Khi nào?

> Trên máy nào?

Ví dụ:
```
Administrator
 ↓
Add User to GG_IT
 ↓
2026-08-26 10:30
```
Hoặc:
```
User01
 ↓
Failed Login
 ↓
DC01
 ↓
2026-08-26 11:20
```

# 30. Những Event quan trọng

Một số Event ID cần đưa vào tài liệu:
| Event | Ý nghĩa                        |
| ----: | ------------------------------ |
|  4624 | Logon thành công               |
|  4625 | Logon thất bại                 |
|  4740 | Account bị Lock                |
|  4720 | Tạo User                       |
|  4722 | Enable User                    |
|  4725 | Disable User                   |
|  4726 | Xóa User                       |
|  4728 | Thêm vào Global Security Group |
|  4732 | Thêm vào Local Security Group  |

Không nhất thiết phải nhớ tất cả, nhưng nên có bảng Event ID trong tài liệu vận hành.

# 31. Audit File Server

Có thể theo dõi:
```
Read
Write
Modify
Delete
```
Nhưng không nên audit mọi thứ một cách vô hạn.

Nếu audit quá rộng:
```
Millions of events
 ↓
Security Log quá lớn
 ↓
Khó phân tích
```
Nên audit các dữ liệu quan trọng.

# 32. Monitoring + Alert

Không chỉ hiển thị Dashboard.

Phải có cảnh báo:
```
DC Disk < 15%
       ↓
Alert

DHCP Scope > 80%
       ↓
Alert

WAN Down
       ↓
Alert

Cisco Port Down
       ↓
Alert

DNS Service Down
       ↓
Alert
```

# 33. Toàn bộ dependency giữa các chương

Đây là phần tôi khuyên bạn đưa vào đầu tài liệu.
```
CHƯƠNG 1
Network
   │
   ▼
CHƯƠNG 2
pfSense
   │
   ▼
CHƯƠNG 3
Cisco VLAN
   │
   ▼
CHƯƠNG 4
AD DS
   │
   ▼
CHƯƠNG 5
DNS
   │
   ▼
CHƯƠNG 6
OU / User / Computer
   │
   ▼
CHƯƠNG 7
DHCP
   │
   ▼
CHƯƠNG 8-11
Group / GPO / Security
   │
   ▼
CHƯƠNG 12
AD CS
   │
   ▼
CHƯƠNG 13
NPS / EAP-TLS
   │
   ▼
CHƯƠNG 14
802.1X / Dynamic VLAN
   │
   ▼
CHƯƠNG 15
pfSense Inter-VLAN Firewall
   │
   ▼
CHƯƠNG 16
Internet Policy
   │
   ▼
CHƯƠNG 17
DHCP/DNS/GPO Advanced
   │
   ├──────────────┐
   ▼              ▼
CHƯƠNG 18      CHƯƠNG 19
File Server       Backup
   │              │
   └──────┬───────┘
          ▼
       CHƯƠNG 20
 Monitoring
 Audit
 Operations
```

# 34. Những thành phần nào thuộc về đâu?

| Thành phần           | Server/Thiết bị       |
| -------------------- | --------------------- |
| Internet             | VNPT                  |
| WAN Gateway          | pfSense               |
| NAT                  | pfSense               |
| Firewall             | pfSense               |
| Inter-VLAN Routing   | pfSense               |
| VLAN                 | Cisco                 |
| Switching            | Cisco                 |
| 802.1X Authenticator | Cisco                 |
| RADIUS               | NPS                   |
| Identity             | AD DS                 |
| DNS                  | Windows DNS           |
| DHCP                 | Windows DHCP          |
| Certificate          | AD CS                 |
| GPO                  | AD                    |
| File Sharing         | FS01                  |
| File Permission      | AD + NTFS             |
| Backup               | Backup Server/NAS     |
| Monitoring           | Monitoring Server     |
| Audit                | Windows/pfSense/Cisco |

# 35. Những nguyên tắc không được phá vỡ
**Nguyên tắc 1 — DC phải có IP tĩnh**
```
DC01 = 192.168.90.10
```

**Nguyên tắc 2 — Domain Client dùng DNS AD**
```
DNS = 192.168.90.10
```
Không dùng public DNS trực tiếp.

**Nguyên tắc 3 — Không dùng Domain Admin cho công việc hàng ngày**

Tách:
```
User Account
Admin Account
```

**Nguyên tắc 4 — Không cấp quyền File Server trực tiếp cho User**

Dùng:
```
User
 ↓
Group
 ↓
Resource
```

**Nguyên tắc 5 — Không bật 802.1X toàn hệ thống ngay**

Phải:
```
1 PC
 ↓
1 Port
 ↓
Certificate
 ↓
NPS
 ↓
802.1X
 ↓
Dynamic VLAN
 ↓
Internet
```
thành công trước.

**Nguyên tắc 6 — Không để tất cả VLAN chung một subnet**

Nếu mục tiêu là VLAN thực sự:
```
VLAN10 ≠ VLAN20 ≠ VLAN30
```
về mặt subnet.

**Nguyên tắc 7 — Backup phải được test restore**
```
Backup ≠ Recovery
```

**Nguyên tắc 8 — Mọi thay đổi production phải có kế hoạch rollback**

Ví dụ thay GPO:
```
Backup GPO
 ↓
Test OU
 ↓
Pilot
 ↓
Production
```

# 36. Quy trình triển khai chuẩn

Nếu biến toàn bộ nội dung thành một tài liệu triển khai thực tế, tôi khuyên chia thành **5 phase**:

**PHASE 1 — Infrastructure**
```
01 Network
02 pfSense
03 Cisco VLAN
```

**PHASE 2 — Microsoft Infrastructure**
```
04 AD DS
05 DNS
06 OU
07 DHCP
08 Users/Groups
09 GPO
10 Security
11 Administration
```

**PHASE 3 — Network Access Control**
```
12 AD CS
13 NPS/EAP-TLS
14 802.1X/Dynamic VLAN
15 pfSense Firewall
16 Internet Policy
17 DHCP/DNS/GPO Advanced
```

**PHASE 4 — Enterprise Services**
```
18 File Server
19 Backup
```

**PHASE 5 — Operations**
```
20 Monitoring
Audit
Incident Response
Maintenance
Documentation
```

# 37. Tiêu chí nghiệm thu toàn bộ hệ thống

Đây là phần rất nên có ở cuối tài liệu.

**Network**
- WAN hoạt động.
- pfSense hoạt động.
- VLAN 10/20/30/40/50/90 hoạt động.
- Trunk Cisco ↔ pfSense hoạt động.
- Inter-VLAN routing hoạt động.
- Firewall policy hoạt động.

**AD**
- DC01 192.168.90.10.
- company.local hoạt động.
- DNS hoạt động.
- Client Join Domain thành công.
- OU hoạt động.
- User/Group hoạt động.
- GPO hoạt động.

**PKI**
- Enterprise CA hoạt động.
- Computer Certificate được cấp.
- NPS Certificate hợp lệ.
- Auto Enrollment hoạt động.

**802.1X**
- Cisco giao tiếp được NPS.
- RADIUS hoạt động.
- Domain Computer xác thực được.
-Non-authorized device bị từ chối.
-Dynamic VLAN hoạt động.

**Firewall**
-User → Internet.
-Accounting → Internet.
-Sales → Internet.
-IT → Internet.
-VLAN isolation hoạt động.
-Server VLAN được bảo vệ.

**File Server**
-FS01 Join Domain.
-SMB hoạt động.
-NTFS permission đúng.
-Share permission đúng.
-AGDLP hoạt động.
-Drive Mapping hoạt động.

**Backup**
-AD System State.
-DHCP.
-GPO.
-AD CS.
-NPS.
-File Server.
-Backup ngoài DC.
-Test Restore.

**Monitoring**
-DC monitoring.
-FS monitoring.
-pfSense monitoring.
-Cisco monitoring.
-ESXi monitoring.
-Alert.
-AD Audit.
-File Audit.
-NPS Audit.

# 38. Một điểm tôi muốn chỉnh lại so với một số hướng dẫn trước

Sau khi nhìn toàn bộ hệ thống như **một kiến trúc duy nhất**, có vài điểm không nên hiểu theo cách đơn giản hóa trước đây:

**1. "Join Domain = có Internet"**

Không phải pfSense tự kiểm tra Join Domain.

Kiến trúc chính xác hơn là:
```
Join Domain
 ↓
Computer Certificate
 ↓
802.1X
 ↓
NPS
 ↓
Cisco Authorization
 ↓
VLAN
 ↓
pfSense
 ↓
Internet
```
Đây là cơ chế **Network Access Control**, chứ không phải pfSense trực tiếp đọc trạng thái AD.

**2. AD không nên là Firewall**

AD quản lý:
- Identity
- Authentication
- Authorization
- Policy

pfSense quản lý:
- Network
- Routing
- Firewall
- NAT

Cisco quản lý:
- Switching
- VLAN
- 802.1X

Đây là sự phân tách trách nhiệm đúng.

**3. Dynamic VLAN và Firewall là hai lớp khác nhau**

Ví dụ:
```
NPS:
"Máy này được vào VLAN 50"

Cisco:
"Đưa máy vào VLAN 50"

pfSense:
"VLAN 50 được phép truy cập những gì?"
```
Ba thành phần không nên bị trộn lẫn.

# 39. Kiến trúc cuối cùng mà tài liệu nên hướng tới
```
                              INTERNET
                                  │
                                 VNPT
                                  │
                           ┌──────▼──────┐
                           │   pfSense   │
                           │             │
                           │ NAT         │
                           │ Firewall    │
                           │ Routing     │
                           │ VPN         │
                           └──────┬──────┘
                                  │
                              802.1Q TRUNK
                                  │
                         ┌────────▼────────┐
                         │ Cisco Catalyst  │
                         │     2960S       │
                         │                 │
                         │ VLAN / 802.1X   │
                         └────────┬────────┘
                                  │
          ┌──────────┬────────────┼───────────┬───────────┐
          │          │            │           │           │
       VLAN 10    VLAN 20      VLAN 30     VLAN 40     VLAN 50
        USER      ACCOUNTING     SALES       WIFI         IT
          │          │            │           │           │
          └──────────┴────────────┴───────────┴───────────┘
                                  │
                               VLAN 90
                                  │
                ┌─────────────────┼──────────────────┐
                │                 │                  │
             DC01               FS01             Monitoring
          .90.10              .90.20
                │                 │
        ┌───────┼───────┐         │
        │       │       │         │
       AD      DNS     DHCP    SMB/NTFS
        │
        ├── OU
        ├── Users
        ├── Groups
        ├── GPO
        │
        ├── AD CS
        │      │
        │      └── Certificate
        │
        └── NPS
               │
               └── RADIUS
                     │
                     ▼
                  802.1X
                     │
                     ▼
               Dynamic VLAN
```

# 40. Kết luận

Nếu gom toàn bộ roadmap thành một câu thì kiến trúc của bạn là:

> **pfSense quản lý biên mạng → Cisco quản lý VLAN và Network Access → Windows AD quản lý Identity → DNS/DHCP cung cấp dịch vụ mạng nội bộ → AD CS cung cấp Certificate → NPS + EAP-TLS xác thực thiết bị → Dynamic VLAN phân loại thiết bị → pfSense áp chính sách Firewall → File Server cung cấp tài nguyên theo AD Group → Backup bảo vệ hệ thống → Monitoring/Audit đảm bảo khả năng vận hành.**

Đây là điểm rất quan trọng: **không nên coi 20 chương là 20 cấu hình độc lập**. Chúng là một chuỗi phụ thuộc:
```
NETWORK
   ↓
AD
   ↓
DNS
   ↓
DHCP
   ↓
GPO
   ↓
PKI
   ↓
NPS
   ↓
802.1X
   ↓
Dynamic VLAN
   ↓
pfSense Firewall
   ↓
File Server
   ↓
Backup
   ↓
Monitoring / Audit
```
Và với hệ thống của bạn, **3 mốc nghiệm thu quan trọng nhất** trước khi đưa vào production là:
```
MỐC 1
PC → DHCP → DNS → Join company.local
                 ↓
              PASS

MỐC 2
PC → Certificate → 802.1X → NPS → Cisco
                                  ↓
                              Dynamic VLAN
                 ↓
              PASS

MỐC 3
Dynamic VLAN → pfSense → Firewall Policy → Internet
                                      ↓
                                   PASS
```
Nếu ba mốc này hoạt động ổn định, các phần **File Server, Backup và Monitoring** có thể triển khai lên trên mà không phải thay đổi kiến trúc lõi.
