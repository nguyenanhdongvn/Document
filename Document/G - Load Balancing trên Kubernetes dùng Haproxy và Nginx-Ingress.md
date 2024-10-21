# Giới thiệu
Thông thường các service trên K8S muốn expose được ra bên ngoài có 2 cách: Node Port hoặc Nginx-Ingress.<br>
Ingress phù hợp với các service dạng http/https, Node Port cũng dùng được cho http/https nhưng thường sử dụng cho các trường hợp còn lại. Ví dụ bạn muốn expose Kafka, VerneMQ ra ngoài thì bạn sẽ dùng Node Port chứ ko nhất thiết phải qua Nginx-Ingress vì cấu hình Node Port đơn giản hơn. <br>
Service trên K8S khi được expose ra sẽ có entry-point là các IP của các node. Do đó để đảm bảo được Load Balancing thì ta sẽ cần một service làm nhiệm vụ LB cho các node đó, thường là Haproxy hoặc Nginx (sẽ được cài trên các K8S node đó). Như vậy Haproxy giải quyết được bài toán LB (Load Balancing). Còn vấn đề HA (High Availability) thì chưa. <br>
Để giải quyết bài toán HA, thì ta sẽ nghĩ đến giải pháp dùng VIP, khi một node down thì sẽ vẫn còn node khác xử lý để không ảnh hưởng service. VIP mình sử dụng là Keepalived, hoặc có thể tham khảo PCS.

## Nginx-Ingress
Nginx-Ingress cài trên K8S được gọi Ingress-Controller, và service của nó thì sẽ được expose ra ngoài K8S bằng Node Port. Ingress Controller đóng vai trò là cầu nối từ bên ngoài internet vào các service bên trong K8S.<br>
Trên K8S có khai báo các Ingress rule, các rule này cấu hình để Nginx-Ingress biết được khi request đến một URL cụ thể thì sẽ cần forward request tới service nào trong K8S.

## Haproxy
Đóng vai trò là LB của hệ thống, đây là sẽ nơi nhận request tới các service, ta sẽ cấu hình cho Haproxy để nó hiểu là với các request nào thì forward tới đâu. Trong trường hợp này thì user (local host) cần truy cập URL http://grafana.dongna.com thì Haproxy sẽ forward request tới Nginx-Ingress, sau đó Nginx-Ingress sẽ xử lý tiếp

## Keepalived
Ta có 3 server cùng có chạy một service, khi 1 server bị down thì user vẫn truy cập được service.<br>

Như trong mô hình lab này, nếu mình chỉ cài Haproxy trên `master1` để làm LB. Khi đó user sẽ cần kết nối tới IP của `master1` để truy cập service. Nếu `master1` down thì service cũng down. Để giải quyết, ta sẽ cài Keepalive trên cả 3 master node và cấu hình một VIP (Virtual IP), tại một thời điểm sẽ chỉ có 1 trong 3 node đứng lên nhận VIP. User sẽ chỉ biết VIP để truy cập service mà không cần quan tâm tới 3 IP của 3 node phía sau là gì. Khi 1 node bị down thì KeepAlived sẽ tự động kiểm tra và chuyển VIP sang một node khác còn active. Như vậy thì Haproxy sẽ cần được cài và cấu hình trên tất cả các node cài keepalive.

# Cài đặt

## Chuẩn bị file server.pem để cấu hình cho haproxy
Tạo folder chứa file `server.pem` trêm 3 Master Node để cấu hình cho haproxy
```
su - sysadmin -c "mkdir /home/sysadmin/ssl/"
```

Từ `cicd` server Copy file `dongna_app.pem`  sang 3 Master Node và đổi tên thành `server.pem`
```
scp /home/sysadmin/ssl/dongna_app.pem sysadmin@192.168.10.11:/home/sysadmin/ssl/server.pem
scp /home/sysadmin/ssl/dongna_app.pem sysadmin@192.168.10.12:/home/sysadmin/ssl/server.pem
scp /home/sysadmin/ssl/dongna_app.pem sysadmin@192.168.10.13:/home/sysadmin/ssl/server.pem
```

## Cài đặt Haproxy và KeepAlived
Trên 3 Master Node, chạy lệnh:
```
yum install haproxy keepalived -y
```

Cấu hình rsyslog cho haproxy tại /etc/rsyslog.d/haproxy.conf để tiện cho việc debug sau này
```
cat <<EOF > /etc/rsyslog.d/haproxy.conf
# Collect log with UDP
$ModLoad imudp
$UDPServerAddress 127.0.0.1
$UDPServerRun 514

# Creating separate log files based on the severity
local0.* /var/log/haproxy-traffic.log
local0.notice /var/log/haproxy-admin.log
EOF
```

Restart rsyslog để apply config
```
systemctl restart rsyslog
```

## Cài đặt Nginx-Ingress bằng helm chart
Trên `cicd` server, tạo thư mục cài đặt

```
mkdir /home/sysadmin/kubernetes_installation/nginx-ingress
```

Download bộ cài về và tạo file config:

```
cd /home/sysadmin/kubernetes_installation/nginx-ingress
helm repo add  nginx-stable https://helm.nginx.com/stable
helm repo update
helm search repo nginx
helm pull nginx-stable/nginx-ingress --version 1.3.2
tar -xzf nginx-ingress-1.3.2.tgz
cp nginx-ingress/values.yaml value-nginx-ingress.yaml
```

Ta sửa lại các parameter của file value mặc định như sau để cho nó chạy Node Port ra 2 port là 30080 và 30443:

```
  service:
    ## The type of service to create for the Ingress Controller.
    type: NodePort
    httpPort:
      ## The custom NodePort for the HTTP port. Requires controller.service.type set to NodePort.
      nodePort: 30080
    httpsPort:
      ## The custom NodePort for the HTTPS port. Requires controller.service.type set to NodePort.
      nodePort: 30443
```

Tạo namespace dành riêng cho nginx-ingress và cài đặt

```
kubectl create ns nginx-ingress
helm -n nginx-ingress install nginx-ingress -f value-nginx-ingress.yaml nginx-ingress
```

Kiểm tra pod vừa tạo trong namespace "nginx-ingress"
```
kubectl get pods -n nginx-ingress
```

Output
```
NAME                                          READY   STATUS    RESTARTS   AGE
nginx-ingress-nginx-ingress-f5b87cc54-hs2lp   1/1     Running   0          7m39s
```

Deploy 1 service `apple_app` để tí nữa test LB và HA
```
cat <<EOF > /home/sysadmin/kubernetes_installation/nginx-ingress/apple.yaml
kind: Pod
apiVersion: v1
metadata:
  name: apple-app
  labels:
    app: apple
spec:
  containers:
    - name: apple-app
      image: hashicorp/http-echo
      args:
        - "-text=THIS_IS_APPLE"

---

kind: Service
apiVersion: v1
metadata:
  name: apple-service
spec:
  selector:
    app: apple
  ports:
    - port: 5678 # Default port for image
EOF
```

Deploy lên K8s
```
kubectl -n nginx-ingress apply -f apple.yaml
```

Tạo file yaml cho ingress rule để trỏ URL `apple.prod.dongna.com` đến apple app bên trong K8s
```
cat <<EOF > /home/sysadmin/kubernetes_installation/nginx-ingress/app.ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:    
  name: apple.prod.dongna.com  
spec:
  ingressClassName: nginx
  rules:
  - host: apple.prod.dongna.com
    http:
      paths:
      - backend:
          service:
            name: apple-service
            port:
              number: 5678
        path: /
        pathType: Prefix
EOF
```

Tạo ingress rule
```
kubectl -n nginx-ingress apply -f app.ingress.yaml
```

Như vậy, khi user truy cập tới URL http://apple.prod.dongna.com/ thì nó sẽ kết nối tới Apple app qua service `apple-service`


# Copy Certificate trong /home/dong/ssl/ từ Local Host lên các Master Node và `cicd` server
Từ Local Host
```
# Create /home/sysadmin/ssl/ folder
ssh sysadmin@cicd 'sudo mkdir /home/sysadmin/ssl/'
ssh sysadmin@master1 'sudo mkdir /etc/haproxy/ssl/'
ssh sysadmin@master2 'sudo mkdir /etc/haproxy/ssl/'
ssh sysadmin@master3 'sudo mkdir /etc/haproxy/ssl/'

# Add permission to write in /home/sysadmin/ssl/ folder
ssh sysadmin@cicd 'sudo chmod o+w /home/sysadmin/ssl/'
ssh sysadmin@master1 'sudo chmod o+w /etc/haproxy/ssl/'
ssh sysadmin@master2 'sudo chmod o+w /etc/haproxy/ssl/'
ssh sysadmin@master3 'sudo chmod o+w /etc/haproxy/ssl/'

# Copy Certificates from Local Host to cicd and Master Node
scp /home/dong/ssl/* sysadmin@cicd:/home/sysadmin/ssl/
scp /home/dong/ssl/dongna_app.pem sysadmin@master1:/etc/haproxy/ssl/server.pem
scp /home/dong/ssl/dongna_app.pem sysadmin@master2:/etc/haproxy/ssl/server.pem
scp /home/dong/ssl/dongna_app.pem sysadmin@master3:/etc/haproxy/ssl/server.pem
```


# Cấu hình
## Cấu hình VIP cho Keepalived
Cấu hình 1 VIP cho cả 3 Master Node, mỗi node sẽ có 1 giá trị priority khác nhau và Keepalived sẽ định kỳ check Haproxy trên từng Master Node xem có bị down không, nếu Haproxy trên Master Node đang làm VIP bị down thì giá trị priority sẽ bị giảm xuống => Master Node khác sẽ đứng lên làm VIP

| Node    |	IP            |	VIP           |	State  | priority |
|---------|---------------|---------------|--------|----------|
| master1 |	192.168.10.11 |	192.168.10.10 |	Master |	100     |
| master2 |	192.168.10.12 |	192.168.10.10	| Backup |	99      |
| master3 |	192.168.10.13 |	192.168.10.10	| Backup |	98      |

Như vậy, `master1` đang là VIP, `master2` và `master3` sẽ backup với priority thấp hơn

Cấu hình keepalived trên `master1`
```
cat <<EOF > /etc/keepalived/keepalived.conf
vrrp_script haproxy-check {
    script "killall -0 haproxy"
    interval 2
    weight 10
}

vrrp_instance kubernetes {
    state MASTER
    priority 100
    interface ens160
    virtual_router_id 61
    advert_int 2
    authentication {
        auth_type AH
        auth_pass dongna
    }
    virtual_ipaddress {
        192.168.10.10
    }

    track_script {
        haproxy-check
    }
}
EOF
```

Restart service keepalived trên `master1` và kiểm tra có VIP mới được tạo ra chưa. Kiểm tra trạng thái của Keepalived
```
systemctl restart keepalived
systemctl status keepalived
```

Output
```
systemctl status keepalived
● keepalived.service - LVS and VRRP High Availability Monitor
     Loaded: loaded (/usr/lib/systemd/system/keepalived.service; disabled; preset: disabled)
     Active: active (running) since Fri 2024-09-27 17:13:46 +07; 13ms ago
   Main PID: 25388 (keepalived)
      Tasks: 2 (limit: 17318)
     Memory: 5.1M
        CPU: 17ms
     CGroup: /system.slice/keepalived.service
             ├─25388 /usr/sbin/keepalived --dont-fork -D
             └─25389 /usr/sbin/keepalived --dont-fork -D

Sep 27 17:13:46 master1 Keepalived_vrrp[25389]: Assigned address 192.168.10.11 for interface ens160
Sep 27 17:13:46 master1 Keepalived_vrrp[25389]: Assigned address fe80::20c:29ff:fe8b:933d for interface ens160
Sep 27 17:13:46 master1 Keepalived_vrrp[25389]: Registering gratuitous ARP shared channel
Sep 27 17:13:46 master1 Keepalived_vrrp[25389]: (kubernetes) removing VIPs.
Sep 27 17:13:46 master1 Keepalived[25388]: Startup complete
Sep 27 17:13:46 master1 Keepalived_vrrp[25389]: (kubernetes) Entering BACKUP STATE (init)
Sep 27 17:13:46 master1 Keepalived_vrrp[25389]: VRRP sockpool: [ifindex(  2), family(IPv4), proto(51), fd(12,13) multicast, address(224.0.0.18)]
Sep 27 17:13:46 master1 systemd[1]: Started LVS and VRRP High Availability Monitor.
Sep 27 17:13:47 master1 Keepalived_vrrp[25389]: Script `haproxy-check` now returning 1
Sep 27 17:13:47 master1 Keepalived_vrrp[25389]: VRRP_Script(haproxy-check) failed (exited with status 1)
```
```
● keepalived.service - LVS and VRRP High Availability Monitor
     Loaded: loaded (/usr/lib/systemd/system/keepalived.service; disabled; preset: disabled)
     Active: active (running) since Fri 2024-09-27 17:13:46 +07; 30s ago
   Main PID: 25388 (keepalived)
      Tasks: 2 (limit: 17318)
     Memory: 5.1M
        CPU: 69ms
     CGroup: /system.slice/keepalived.service
             ├─25388 /usr/sbin/keepalived --dont-fork -D
             └─25389 /usr/sbin/keepalived --dont-fork -D

Sep 27 17:13:53 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:53 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:53 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:53 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: (kubernetes) Sending/queueing gratuitous ARPs on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
```

Lúc này check lại IP của node sẽ thấy VIP 192.168.10.10 mới được tạo ra:
```
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:8b:93:3d brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.10.11/24 brd 192.168.10.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet 192.168.10.10/32 scope global ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe8b:933d/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

Cấu hình keepalived trên `master2`
```
cat <<EOF > /etc/keepalived/keepalived.conf
vrrp_script haproxy-check {
    script "killall -0 haproxy"
    interval 2
    weight 10
}

vrrp_instance kubernetes {
    state MASTER
    priority 99
    interface ens160
    virtual_router_id 61
    advert_int 2
    authentication {
        auth_type AH
        auth_pass dongna
    }
    virtual_ipaddress {
        192.168.10.10
    }

    track_script {
        haproxy-check
    }
}
EOF
```
Restart service keepalived trên `master2`. Kiểm tra trạng thái của Keepalived
```
systemctl restart keepalived
systemctl status keepalived
```

Cấu hình keepalived trên `master3`
```
cat <<EOF > /etc/keepalived/keepalived.conf
vrrp_script haproxy-check {
    script "killall -0 haproxy"
    interval 2
    weight 10
}

vrrp_instance kubernetes {
    state MASTER
    priority 98
    interface ens160
    virtual_router_id 61
    advert_int 2
    authentication {
        auth_type AH
        auth_pass dongna
    }
    virtual_ipaddress {
        192.168.10.10
    }

    track_script {
        haproxy-check
    }
}
EOF
```
Restart service keepalived trên `master3`. Kiểm tra trạng thái của Keepalived
```
systemctl restart keepalived
systemctl status keepalived
```

Ta đã cấu hình 3 Master Node có chung VIP 192.168.10.10, khi Haproxy trên keepalived-master bị down (hoặc node keepalived-master bị down) thì VIP sẽ được chuyển sang keepalived-backup (dựa vào tham số priority trong cấu hình keepalived).











## Cấu hình Haproxy làm SSL Termination (SSL Offload)
Để cấu hình LB cho các Master Node trong K8s cluster thì ta tạo rule để round-robin các request tới các Master Node tại Node Port của Nginx-Ingress. Phần còn lại Nginx-Ingress sẽ phân tích URL của request và forward tiếp tới các service bên trong K8s dựa vào các ingress-rule.

Ta cấu hình Haproxy cho cả 3 Master Node giống nhau vì chỉ node nào đang nhận VIP sẽ làm nhiệm vụ LB, còn các node còn lại ở trạng thái Backup sẵn sàng thay khi Master Node down.

Cấu hình Haproxy ở file config /etc/haproxy/haproxy.cfg. Ở đây mình có 2 backend, một default là trỏ mọi request tới Nginx-Ingress của K8s, một là check hostname nếu là rancher.monitor.dongna.com thì forward tới `rancher` server

```
cat <<EOF > /etc/haproxy/haproxy.cfg
global
    log         127.0.0.1:514 local0

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
    stats socket *:1999 level admin
    stats socket /var/run/haproxy.sock mode 600 level admin
    
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

listen stats
    bind *:8085
    stats enable
    stats uri /stats
    stats realm HAProxy-04\ Statistics
    stats auth admin:password
    stats admin if TRUE

backend per_ip_and_url_rates
    stick-table type binary len 8 size 1m expire 24h store http_req_rate(24h)

backend per_ip_rates
    stick-table type ip size 1m expire 24h store gpc0,gpc0_rate(30s)

frontend frontend_ssl_443
        bind :80
        bind *:443 ssl crt /etc/haproxy/ssl/server.pem
        mode http
        option httpclose
        option forwardfor
        http-request add-header X-Forwarded-Proto https
        cookie  SRVNAME insert indirect nocache
        default_backend backend_ingress
        #default_backend backend_rancher

        acl rancher hdr_dom(host) -i rancher.dongna.com
        use_backend backend_rancher if rancher

backend backend_ingress
        mode    http
        stats   enable
        stats   auth username:password
        balance roundrobin
        server  worker1 192.168.10.14:30080 cookie p1 weight 1 check inter 2000
        server  worker2 192.168.10.15:30080 cookie p1 weight 1 check inter 2000
        server  worker3 192.168.10.16:30080 cookie p1 weight 1 check inter 2000


backend backend_rancher
        mode    http
        stats   enable
        stats   auth username:password
        balance roundrobin
        server  rancher 192.168.10.19:6860 cookie p1 weight 1 check inter 2000
EOF
```

Kiểm tra syntax và restart Haproxy trên 3 Master Node
```
haproxy -c -f /etc/haproxy/haproxy.cfg
systemctl restart haproxy.service
```

## Giải thích chi tiết config của Haproxy
```
...
frontend frontend_ssl_443
        bind :80
        bind *:443 ssl crt /home/sysadmin/ssl/server.pem
        mode http
        option httpclose
        option forwardfor
        http-request add-header X-Forwarded-Proto https
        cookie  SRVNAME insert indirect nocache
        default_backend backend_ingress

        acl rancher hdr_dom(host) -i rancher.dongna.com
        use_backend backend_rancher if rancher
...
```
Trong đó:
- **frontend frontend_ssl_443**: Chỉ ra một frontend có tên là frontend_ssl_443
- **bind :80**: Chỉ ra frontend sẽ listen ở port 80
- **bind *:443 ssl crt /home/sysadmin/ssl/server.pem**:
  - **bind *:443**: Chỉ ra frontend sẽ listen ở port 443 ở tất cả các network interface (chú ý dấu * trước port 443)
  - **SSL**: là bật tính năng SSL Termination cho listener này.
  - **CRT**: chỉ ra đường dẫn tới file SSL-Certificate, ta đã thực hiện tạo ở bước trước cần copy lên máy chủ cài haproxy và cấu hình đường dẫn vào đây.
- **http-request add-header X-Forwarded-Proto https**: Thêm https header và cuối HTTPS request
- **default_backend backend_ingress**: Cấu hình mặc định request nếu ko match với rule ALC nào thì sẽ vào backend là backend_ingress, đây là rule để mặc định sẽ kết nối tới các app trên K8S thông qua Nginx-Ingress
- **acl rancher hdr_dom(host) -i rancher.monitor.dongna.com**: Tạo điều kiện check rancher nếu host request tới trùng với địa chỉ "rancher.monitor.dongna.com"
- **use_backend backend_rancher if rancher**: Nếu điều kiện rancher là đúng thì trỏ tới backend là backend_rancher

Cấu hình backend_ingress: Thực hiện load balancing request tới 3 k8s worker node, port 30080 là Node Port của Nginx-Ingress

```
backend backend_ingress
        mode    http
        stats   enable
        stats   auth username:password
        balance roundrobin
        server  worker1 192.168.10.14:30080 cookie p1 weight 1 check inter 2000
        server  worker2 192.168.10.15:30080 cookie p1 weight 1 check inter 2000
        server  worker3 192.168.10.16:30080 cookie p1 weight 1 check inter 2000
```

Cấu hình backend_rancher: Thực hiện forward kết nối tới rancher-server cài trên trên rancher ở địa chỉ IP 192.168.10.19 và port là 6860 (lưu ý đây là port HTTP của rancher)
```
backend backend_rancher
        mode    http
        stats   enable
        stats   auth username:password
        balance roundrobin
        server  rancher 192.168.10.19:6860 cookie p1 weight 1 check inter 2000
```

Khai báo host trên client (local host)
Trong bài viết trước mình đã mô tả bước này, cần khai host cho ứng dụng ở client như sau:
```
192.168.10.10 apple.demo.dongna.com
192.168.10.10 rancher.dongna.com
```

















Ta cũng đã tạo SSL certificate cho các application bên trong K8s tại [đây](https://github.com/nguyenanhdongvn/Document/blob/main/Document/G%20-%20H%C6%B0%E1%BB%9Bng%20d%E1%BA%ABn%20t%E1%BA%A1o%20certificate%20cho%20%E1%BB%A9ng%20d%E1%BB%A5ng%20tr%C3%AAn%20K8S%20d%C3%B9ng%20OpenSSL.md)

# Cấu hình ở user (local host) và kiểm tra truy cập
Để truy cập được vào Rancher (rancher.monitor.dongna.com/) và Apple app (apple.prod.dongna.com/), user sẽ chỉ quan tâm tới VIP chứ ko cần biết IP của 3 Master Node

User sẽ cần cấu hình file host trên máy user<br>
C:\Windows\System32\drivers\etc\hosts (window)
```
192.168.10.10 rancher.dongna.com
192.168.10.10 apple.prod.dongna.com
```

# Kiểm tra tính sẵn sàng (High Availability)
Hiện tại `master1` đang là master của cụm VIP và đang nhận VIP là 192.168.10.10, cũng là server đang thực hiện load balancing cho hệ thống.
```
[root@master1 ssl]# ip a |grep 192.168.10.10
    inet 192.168.10.10/32 scope global ens160
```

Stop Haproxy trên `master1` xem điều gì sẽ xảy ra
```
systemctl stop haproxy.service
```

Check status Keepalived trên `master1`
```
systemctl status keepalived
```

Output
```
[root@master1 ~]# systemctl status keepalived
● keepalived.service - LVS and VRRP High Availability Monitor
     Loaded: loaded (/usr/lib/systemd/system/keepalived.service; disabled; preset: disabled)
     Active: active (running) since Fri 2024-09-27 17:13:46 +07; 13min ago
   Main PID: 25388 (keepalived)
      Tasks: 2 (limit: 17318)
     Memory: 5.1M
        CPU: 1.389s
     CGroup: /system.slice/keepalived.service
             ├─25388 /usr/sbin/keepalived --dont-fork -D
             └─25389 /usr/sbin/keepalived --dont-fork -D

Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:13:58 master1 Keepalived_vrrp[25389]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 27 17:21:59 master1 Keepalived_vrrp[25389]: Script `haproxy-check` now returning 0
Sep 27 17:21:59 master1 Keepalived_vrrp[25389]: VRRP_Script(haproxy-check) succeeded
Sep 27 17:21:59 master1 Keepalived_vrrp[25389]: (kubernetes) Changing effective priority from 100 to 110
Sep 27 17:27:09 master1 Keepalived_vrrp[25389]: Script `haproxy-check` now returning 1
Sep 27 17:27:09 master1 Keepalived_vrrp[25389]: VRRP_Script(haproxy-check) failed (exited with status 1)
Sep 27 17:27:09 master1 Keepalived_vrrp[25389]: (kubernetes) Changing effective priority from 110 to 100
```

Ta thấy output gồm "VRRP_Script(haproxy-check) failed" và "VRRP_Instance(kubernetes) Changing effective priority from 110 to 100".<br>
Tham số Priority ban đâu trên 3 Node là 100, 99 và 98. Kèm theo đó, keepalived check service haproxy vẫn UP thì priority cộng thêm 10. Tức là trường hợp Haproxy trên 3 Node đều UP thì priority sẽ là 110, 109 và 108.
Khi Haproxy trên `master1` bị down, thì priority chỉ còn 100, lúc này `master2` có priority là 109 sẽ dc đẩy lên làm master và sẽ nhận VIP.


Check status Keepalived trên `master2`
```
systemctl status keepalived
```

Output
```
● keepalived.service - LVS and VRRP High Availability Monitor
     Loaded: loaded (/usr/lib/systemd/system/keepalived.service; disabled; preset: disabled)
     Active: active (running) since Sun 2024-09-29 03:27:26 +07; 41min ago
   Main PID: 158745 (keepalived)
      Tasks: 2 (limit: 17318)
     Memory: 2.2M
        CPU: 9.558s
     CGroup: /system.slice/keepalived.service
             ├─158745 /usr/sbin/keepalived --dont-fork -D
             └─158746 /usr/sbin/keepalived --dont-fork -D

Sep 29 04:08:06 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 29 04:08:06 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 29 04:08:06 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 29 04:08:06 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 29 04:08:11 master2 Keepalived_vrrp[158746]: (kubernetes) Sending/queueing gratuitous ARPs on ens160 for 192.168.10.10
Sep 29 04:08:11 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 29 04:08:11 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 29 04:08:11 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 29 04:08:11 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
Sep 29 04:08:11 master2 Keepalived_vrrp[158746]: Sending gratuitous ARP on ens160 for 192.168.10.10
```

Check IP trên `master2`
```
[root@master2 ssl]# ip a |grep 192.168.10.10
    inet 192.168.10.10/32 scope global ens160
```

Lúc này user truy cập vào VIP và vẫn truy cập application bình thường
