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
## Cài đặt Haproxy và KeepAlived
Trên 3 Master Node, chạy lệnh:
```
sudo yum install haproxy keepalived -y
```

Cấu hình rsyslog cho haproxy tại /etc/rsyslog.d/haproxy.conf để tiện cho việc debug sau này
```
# Collect log with UDP
$ModLoad imudp
$UDPServerAddress 127.0.0.1
$UDPServerRun 514

# Creating separate log files based on the severity
local0.* /var/log/haproxy-traffic.log
local0.notice /var/log/haproxy-admin.log
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
helm pull nginx-stable/nginx-ingress --version 0.13.0
tar -xzf nginx-ingress-0.13.0.tgz
cp nginx-ingress/values.yaml value-nginx-ingress.yaml
```

Ta sửa lại các parameter của file value mặc định như sau để cho nó chạy Node Port ra 2 port là 30080 và 30443:

```
    type: NodePort
    ## The custom NodePort for the HTTP port. Requires controller.service.type set to NodePort.
    nodePort: 30080
    ## The custom NodePort for the HTTPS port. Requires controller.service.type set to NodePort.
    nodePort: 30443
```

Tạo namespace dành riêng cho nginx-ingress và cài đặt

```
kubectl create ns nginx-ingress
helm -n nginx-ingress install nginx-ingress -f value-nginx-ingress.yaml nginx-ingress
```

Output
```
[sysadmin@cicd nginx-ingress]$  kubectl -n nginx-ingress get pods
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

```

Lúc này check lại IP của node sẽ thấy VIP 192.168.10.10 mới được tạo ra:
```

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

## Cấu hình Haproxy
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

        acl rancher hdr_dom(host) -i rancher.monitor.dongna.com
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

Kiểm tra syntax và restart Haproxy
```
haproxy -c -f /etc/haproxy/haproxy.cfg
systemctl restart haproxy.service
```

Ta cũng sẽ tạo SSL certificate cho các application bên trong K8s tại [đây](https://github.com/nguyenanhdongvn/Document/blob/main/Document/G%20-%20H%C6%B0%E1%BB%9Bng%20d%E1%BA%ABn%20t%E1%BA%A1o%20certificate%20cho%20%E1%BB%A9ng%20d%E1%BB%A5ng%20tr%C3%AAn%20K8S%20d%C3%B9ng%20OpenSSL.md)

Kết quả sinh ra lưu vào file /etc/haproxy/ssl/server.pem để dùng cho cấu hình haproxy.

# Cấu hình ở user (local host) và kiểm tra truy cập
Để truy cập được vào Rancher (rancher.monitor.dongna.com/) và Apple app (apple.prod.viettq.com/), user sẽ chỉ quan tâm tới VIP chứ ko cần biết IP của 3 Master Node

User sẽ cần cấu hình file host trên máy user<br>
C:\Windows\System32\drivers\etc\hosts (window)
```
192.168.10.10 rancher.monitor.dongna.com
192.168.10.10 apple.prod.dongna.com
```

# Kiểm tra tính sẵn sàng (High Availability)
Hiện tại `master1` đang là master của cụm VIP và đang nhận VIP, cũng là server đang thực hiện load balancing cho hệ thống.
```

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
```

Ta thấy output gồm "VRRP_Script(haproxy-check) failed" và "VRRP_Instance(kubernetes) Changing effective priority from 110 to 100".<br>
Tham số Priority ban đâu trên 3 Node là 100, 99 và 98. Kèm theo đó, keepalived check service haproxy vẫn UP thì priority cộng thêm 10. Tức là trường hợp Haproxy trên 3 Node đều UP thì priority sẽ là 110, 109 và 108.
Khi Haproxy trên `master1` bị down, thì priority chỉ còn 100, lúc này `master2` có priority là 109 sẽ dc đẩy lên làm master và sẽ nhận VIP.


Check status Keepalived trên `master2`
```
```

Check IP trên `master2`
```
```

Lúc này user truy cập vào VIP và vẫn truy cập application bình thường
