# Cấu hình Loadbalancing bằng NGINX trên Ubuntus Server
## Nội dung
- **I. Cài đặt gói BIND và NGINX**
- **II. Cấu hình ip tĩnh trên Server Loadbalnacer, 2 nodes server, và 1 client**
    - **1. Cấu hình trên Loadbalancer**
    - **2. Cấu hình trên Nodes**
    - **3. Cấu hình trên Client**
- **III. Cấu hình DNS trên Server Loadbalancer**
    - **1. Cấu hình BIND**
    - **2. Khai báo zone**
    - **3. Tạo file record của các zone**
    - **4. Khởi động dịch vụ DNS**
- **III. Cấu hình NGINX**
    - **1. Cấu hình Loadbalancer Server**
    - **2. Cấu hình trên các Nodes**
    - **3. Khởi động lại dịch vụ NGINX**

## I. Cài đặt BIND và NGINX
### 1. Chuẩn bị 
- Hãy đảm bảo máy ảo có kết nối mạng đến bên ngoài.
![](./nat.png)
- Tiến hành cài gói BIND
- ```sudo apt update```
- ```sudo apt install bind9 bind9utils bind9-doc```
- ```sudo nano /etc/default/named```
- Thêm -4 vào ```OPTIONS="-u bind -4"``` nhằm đảm bảo rằng nó chỉ sử dụng IPv4, giúp tránh các vấn đề tiềm ẩn liên quan đến Ipv6.
