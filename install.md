# Project5
# Cài đặt và cấu hình Keepalived và Haproxy trên CentOS

#  Nội dung:
## Giới thiệu về Keepalived

- Keepalived là một phần mềm định tuyến, được viết bằng ngôn ngữ C. Chương trình keepalived cho phép nhiều máy tính cùng chia sẻ một địa chỉ IP ảo với nhau theo mô hình Active – Passive (ta có thể cấu hình thêm một chút để chuyển thành mô hình Active – Active).

- Khi người dùng cần truy cập vào dịch vụ, người dùng chỉ cần truy cập vào địa chỉ IP ảo dùng chung này thay vì phải truy cập vào những địa chỉ IP thật của các thiết bị kia.

- Một số đặc điểm của phần mềm Keepalived:

  - Keepalived không đảm bảo tính ổn định của dịch vụ chạy trên máy chủ, nó chỉ đảm bảo rằng sẽ luôn có ít nhất một máy chủ chịu trách nhiệm cho IP dùng chung khi có sự cố xảy ra.
  - Keepalived thường được dùng để dựng các hệ thống HA (High Availability) dùng nhiều router/firewall/server để đảm bảo hệ thống được hoạt động liên tục.
  - Keepalived dùng giao thức VRRP (Virtual Router Redundancy Protocol) để liên lạc giữa các thiết bị trong nhóm.
  
## Giới thiệu về giao thức VRRP

- Virtual router đại diện cho một nhóm thiết bị sẽ có một virtual IP và một đỉa chỉ MAC (Media Access Control) đặc biệt là `00-00-5E-00-01-XX`. Trong đó, `XX` là số định danh của router ảo – Virtual Router Identifier (VRID), mỗi virtual router trong một mạng sẽ có một giá trị VRID khác nhau. Vào mỗi thời điểm nhất định, chỉ có một router vật lý dùng địa chỉ MAC ảo này. Khi có ARP request gởi tới virtual IP thì router vật lý đó sẽ trả về địa chỉ MAC này.

- Các router vật lý sử dụng chung VIP phải liên lạc với nhau bằng địa chỉ multicast `224.0.0.18` bằng giao thức VRRP. Các router vật lý sẽ có độ ưu tiên (priority) trong khoảng từ 1 – 254, và router có độ ưu tiên cao nhất sẽ thành Master, các router còn lại sẽ thành các Slave/Backup, hoạt động ở chế độ chờ.
## HAproxy
- HAproxy là một phần mềm cân bằng tải tcp/http mã nguồn mở được sử dụng rộng rãi hiện nay.
- Mô tả giải pháp cân bằng tải sử dụng HAProxy
    -Cân bằng tải là một phương pháp phân phối khối lượng truy cập trên nhiều máy chủ nhằm tối ưu hóa tài nguyên hiện có đồng thời tối    đa hóa thông lượng, giảm thời gian đáp ứng và tránh tình trạng quá tải cho một máy chủ.

    - HAProxy (High Availability Proxy) là một giải pháp mã nguồn mở về cân bằng tải có thể dùng cho nhiều dịch vụ chạy trên nền TCP (Layer 4), phù hợp với việc cân bằng tải với giao thức HTTP giúp ổn định phiên kết nối và các tiến trình Layer 7.

    - Cân bằng tải ở Layer 4 chỉ thích hợp cho việc bạn có các webserver có cùng một ứng dụng.
    - Cân bằng tải ở Layer 7 có thể phân tải cho các ứng dụng trên một webserver có nhiều ứng dụng cùng domain.

- Một số lợi ích khi sử dụng phương pháp cân bằng tải:
    - Tăng khả năng đáp ứng, tránh tình trạng quá tải
    - Tăng độ tin cậy và tính dự phòng cao
    - Tăng tính bảo mật cho hệ thống.
 
# Mô hình
-  Sử dụng 2 máy ảo CentOS7  địa chỉ ip lần lượt là `10.10.10.17` và `10.10.10.18`
<img src="https://i.imgur.com/MGiBEGl.jpg">

# Cài đặt
- Cài đặt Keepalived, HAproxy và Httpd:
```sh
#yum -y install keepalived
#yum -y install haproxy
#yum -y install httpd  
```
- Cấu hình **Keepalived**
```sh
#nano /etc/keepalived/keepalived.conf
```
- Nội dung: 
```sh
! Configuration File for keepalived
global_defs {
   notification_email {
     bmng@gmail.com
   }
   smtp_server 8.8.8.8
   smtp_connect_timeout 30
   router_id LVS_MASTER
}
vrrp_script chk_httpd {
        script "killall -0 httpd"       # Xác minh Pid có tồn tại không 
        interval 2                      # Kiểm tra mỗi 2s
        weight 2                        # 2 khi điều kiện đúng
}
vrrp_instance VI_1 {
    state MASTER         # Máy còn lại đặt giá trị = BACKUP
    interface eth0
    virtual_router_id 51
    priority 100         # Máy còn lại đặt giá trị = 99; 
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass Abc@123  #password phải giống nhau trên cả 2 server
    }
    virtual_ipaddress {
        10.10.10.50     # VIP
    }
    track_script {
        chk_httpd
    }
}
```
- Enable Iptable rules trên MASTER và SLAVE SERVER
```sh
#iptables -I INPUT -p vrrp -j ACCEPT
```
- Khởi động lại dịch vụ:
```sh
#service keepalived restart 
#service httpd restart
```
**Giải thích**
- 2 server sẽ tạo ra một địa chỉ VIP `10.10.10.50`. 
- Sv1 ở trạng thái MASTER, Sv2 ở trạng thái BACKUP. Toàn bộ các request gửi đến sẽ thông qua Sv1 trước.
- Ở cả Sv1 và Sv2 đều có track-script: nghĩa là cả 2 Sv sẽ cùng check proccess ID (PID) của chương trình HTTPD. Nếu dịch vụ trên Sv1 bị ngưng. Thì Keepalived sẽ trừ trọng số (priority = 100 - 2 = 98)  trọng số này nhỏ hơn mức đặt ở Sv2. Keepalived sẽ chuyển trạng thái của Sv1 từ MASTER sang BACKUP và ngược lại ở Sv2. 

## Cấu hình HAproxy

- Thực hiện trên cả Sv1 và Sv2:
```sh
#nano /var/haproxy/haproxy.cfg
```
- Nội dung:
```sh
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  main *:80
    default_backend             app
#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static
    balance     roundrobin
    server      static 10.10.10.50:80 check
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    server  app1 10.10.10.17:8080 check  //weight 2 => app1 received 2 req -> app2 1 req
    server  app2 10.10.10.18:8080 check  //weight 1 => app2 = 50% app1
```
- Khởi động lại hệ thống: 
```sh 
#service keepalived restart 
```
## Kiểm tra: 

- Server HAproxy ở Sv1 ngừng hoạt động: Truy cập vào địa chỉ `10.10.10.50` vẫn thực hiện được. ( HAproxy ở Sv2 vẫn điều hướng các request chia đều cho các websv)
- Web server ở Sv2 bị tắt. Tất cả các request vẫn được tiếp nhận bình thường. Nhưng đích đến sẽ được chuyển hết về websv ở Sv1.







