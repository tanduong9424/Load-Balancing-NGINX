# Cấu hình Loadbalancing bằng NGINX trên CentOS
## Nội dung
- **I. Cài đặt gói BIND và NGINX**
- **II. Cấu hình ip tĩnh**
    - **1. Cấu hình IP tĩnh trên Loadbalancer**
    - **2. Cấu hình IP tĩnh trên các Node**
    - **3. Cấu hình IP tĩnh trên Client**
- **III. Cấu hình DNS trên Server Loadbalancer**
    - **1. Cấu hình BIND**
    - **2. Khai báo zone**
    - **3. Tạo file record của các zone**
    - **4. Khởi động dịch vụ DNS**
- **III. Cấu hình NGINX**
    - **1. Cấu hình Loadbalancer Server**
    - **2. Cấu hình trên các node**
    - **3. Khởi động lại dịch vụ NGINX**

## I. Cài đặt BIND và NGINX
### 1. Chuẩn bị 
- Hãy đảm bảo máy ảo có kết nối mạng đến bên ngoài.
![](./nat.png)
### 2. Tiến hành cài gói BIND
- Tiến hành cài gói BIND như sau:
- ```sudo yum update -y```
- ```yum install bind -y```, để kiểm tra cài đặt thành công ta gõ ```rpm -qa| grep bind```.
- Chỉ cần cài đặt trên Load blancer server thôi, không cần cài trên 2 node.
### 3. Tiến hành cài đạt gói NGINX
- Nginx không có sẵn trong kho lưu trữ mặc định của CentOS, vì vậy bạn cần thêm kho EPEL (Extra Packages for Enterprise Linux)
- ```sudo yum install epel-release``` tiếp theo ta cài đặt nginx ```sudo yum install nginx -y```, kiểm tra việc cài đặt bằng  ```rpm -qa| grep nginx```
- Ta tiến hành tắt tường lửa đi để dễ dàng cho việc demo:
```
systemctl stop firewalld
systemctl disable firewalld
```
- Làm tương tự trên 2 node.
## II. Cấu hình IP tĩnh
```
Server Load Balancer: 192.168.1.1/24
Node 1: 192.168.1.2/24
Node 2: 192.168.1.3/24
Client 1: 192.168.1.10/24
Client 2: 192.168.1.20/24 (không bắt buộc)
```
![](tolopy.png)
### 1. Cấu hình IP tĩnh trên Loadbalancer
- Ở dây ta nên tiến hành chuyển card mạng sang host only:
  
![](hostonly.png)
- Sau đó ta cấu hình IP tĩnh như hình dưới:
  
![](StaticIP.png)

- Sau khi cấu hình xong ta có thể khởi động NGINX và truy cập vào 192.168.1.1 bằng firefox,chorme,... để có thể thấy nginx hoạt động như dưới:
- ```
    sudo systemctl start nginx
    sudo systemctl enable nginx
    sudo systemctl status nginx
  ```
![](defaultnginx.png)

### 2. Cấu hình IP tĩnh tNGINX load lên.
- Tiến hành tạo block cho serve sgu.edu.vn ```touch /etc/nginx/conf.d/sgu.edu.vn.conf```, sau đó cấu hình như sau :
```sh
upstream backend {
	#least_conn;
	#slow_start;
	#ip_hash;
	#server ns2.sgu.edu.vn weight=3;
	#server ns3.sgu.edu.vn weight=1 ;
	server ns2.sgu.edu.vn; #đây node thứ nhất để chịu tải .Mặc định thuật toán điều hướng sẽ là Round Robin
	server ns3.sgu.edu.vn; #đây node thứ hai để chịu tải
}

server {
	listen       80;
	server_name  sgu.edu.vn ns1.sgu.edu.vn;

	location / {	#thêm block location để trỏ đến upstream ở trên chứa 2 node
		proxy_pass http://backend;
	}
}
```
## Giải thích về cách tham số và thuật toán:
- ```weight``` :thông số càng lớn , Round Robin điều hướng về server đó càng nhiều, ví dụ server1 weight=3, server2 weight=1 , khi có 4 request, 3 request sẽ được điều hướng về server 1, 1 request sẽ được đưa về server2 .Tuy nhiên trước khi weight được áp dụng nginx sẽ thực hiện vòng phân phối đầu tiên theo cách cơ bản của round-robin (1 lần đến mỗi server) để đảm bảo rằng tất cả các server trong nhóm backend đã sẵn sàng và hoạt động bình thường.
- ```ip_hash```: Nginx lấy ip của client đến một backend server cụ thể, đảm bảo rằng các yêu cầu từ cùng một client luôn được xử lý bởi cùng một server.
- ```least_conn``` : server có ít kết nối nhất thì được điều hướng đến
- ```slow_start``` :VD trong 30s thì nó không đổ dồn 100 ngườingười request và mà nó sẽ tăng chầm chậm dần lên từ 0->1->2.. rồi sao cho trong 30s đó nó sẽ trở về với khả nằng đáp ứng request ban đầu là 100 chẳng hạn.
- ```down``` : đánh dấu server đó không hoạt động.
 
- Kiểm tra lại cấu hình bằng ```nginx -t```
- Restart lại 	```sudo systemctl restart nginx```
- Nếu có lỗi ta tiến hành ```journalctl -xe nginx``` để tìm lỗi nếu là lỗi chính tả. Nếu không có lỗi chính tả ta tiến hành ```sudo lsof -i :80``` để kiểm tra các tiến trình đang dùng chung port 80 với Nginx, nếu có hãy kết thúc bằng câu lệnh ```kill -9 PID``` với PID là ID của tiến trình đang dùng port 80 đã nói trên. Sau đó tiến hành restart lại Nginx.

#### Tiếp đến ta cấu hình web ở trên 2 node
- Tiến hành tạo block cho serve sgu.edu.vn trên node thứ nhất```touch /etc/nginx/conf.d/sgu.edu.vn.conf```, sau đó cấu hình trên Node thứ nhất như sau :
``` sh
server {
	listen       80;
	server_name  ns2.sgu.edu.vn;
	root         /usr/share/nginx/html;
	index	index.html; # thêm dòng này để chỉ đến file index
}
```
- Làm tương tự trên node thứ 2 và cấu hình như sau:
```sh
server {
	listen       80;
	server_name  ns3.sgu.edu.vn;
	root         /usr/share/nginx/html;
	index	index.html;
}
```

- ```sudo nginx –t```
- ```sudo systemctl restart nginx```
- Cuối cùng ta sẽ truy cập đến đường dẫn đã trỏ lúc này để bắt đầu phát triển web
- ```cd /usr/share/nginx/html```
- Tiến hành phát triển file index.html
- ```sudo gedit index.html```
- Sau đó có thể code web theo ý thích chẳng hạn ở đây nhóm mình có 1 đoạn code nhỏ cho mọi người demo, các bạn có thể copy và paste vào file index.html này.
 [File cho node thứ nhất](./index1.html)
[File cho node thứ hai](./index2.html)

- Sau khi thực hiện xong ta restart NGINX để hoạt động:
```
sudo nginx -t
sudo systemctl restart nginx
```

- Trên client ta có thể dùng trình duyệt web để truy cập đến sgu.edu.vn, mỗi lần request, server sẽ điều hướng đến 1 node khác nhau và thể hiện một giao diện web khác nhau.
- Đến đây là đã kết thúc phần tài liệu tham khảo về Load Balancing sử dụng NGINX trên CentOS của nhóm chúng em, cảm ơn mọi người đã quan tâm.
