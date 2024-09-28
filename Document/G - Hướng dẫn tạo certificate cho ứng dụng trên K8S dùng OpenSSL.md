# Tạo Certificate Authority
Để có Certificate xịn thì bạn phải mua của các CA. Ta cũng có thể tự tạo một CA ở trên Local và import vào trình duyệt để trình duyệt hiểu là CA (mà chúng ta tạo ra) là một nhà cung cấp CA tin cậy. Sau đó, các Certificate được cấp bởi CA này sẽ được coi là validate.

# Tạo private key cho CA (bắt buộc phải nhập pass phrase)
```
openssl genrsa -des3 -out rootCA.key 2048
```

# Tạo file pem từ file private key (nhập pass của rootCA đã tạo bên trên)
```
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1825 -out rootCA.pem
```
```
Enter pass phrase for rootCA.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:VN
State or Province Name (full name) []:HCM
Locality Name (eg, city) [Default City]:HCM
Organization Name (eg, company) [Default Company Ltd]:DongNA_CA
Organizational Unit Name (eg, section) []:DongNA_CA_Unit
Common Name (eg, your name or your server's hostname) []:*.dongna.com
Email Address []:
```
Ta sẽ có 2 file .key và .pem cho rootCA
```
sysadmin@master1:~/ssl$ ll
total 8
-rw-------. 1 sysadmin sysadmin 1862 Sep 23 22:33 rootCA.key
-rw-r--r--. 1 sysadmin sysadmin 1354 Sep 23 22:35 rootCA.pem
```
# Tạo SSL Certificate cho ứng dụng web ở local
Đầu tiên ta tạo một file openssl.cnf để cấu hình thêm thông tin SAN như sau:
```
cat <<EOF > /home/sysadmin/ssl/openssl.cnf
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
[req_distinguished_name]
countryName = VN
countryName_default = VN
stateOrProvinceName = HCM
stateOrProvinceName_default = HCM
localityName = HCM
localityName_default = HCM
organizationalUnitName = DongNA_DEVOPS
organizationalUnitName_default = DongNA_DEVOPS
commonName = *.dongna.com
commonName_max = 64
[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.monitor.dongna.com
DNS.2 = *.prod.dongna.com
DNS.3 = *.demo.dongna.com
EOF
```
Ở đây mình sẽ tạo SSL Certcificate cho các app của mình sử dụng 3 subdomain là *.monitor.dongna.com, *.prod.dongna.com và *.demo.dongna.com <br>

Tiếp theo ta tạo file key:
```
openssl genrsa -out dongna_app.key 2048
```

Sau đó ta tạo file Sigining Request từ file key và file config trên:
```
openssl req -new -out dongna_app.csr -key dongna_app.key -config openssl.cnf
```

Kết quả sinh ra file dongna_app.csr
```
sysadmin@master1:~/ssl$ ll
total 20
-rw-r--r--. 1 root     root     1102 Sep 23 22:55 dongna_app.csr
-rw-------. 1 root     root     1704 Sep 23 22:54 dongna_app.key
-rw-r--r--. 1 sysadmin sysadmin  645 Sep 23 22:53 openssl.cnf
-rw-------. 1 sysadmin sysadmin 1862 Sep 23 22:33 rootCA.key
-rw-r--r--. 1 sysadmin sysadmin 1354 Sep 23 22:35 rootCA.pem
```

Rồi tới bước quan trọng nhất là mang đơn đi đóng dấu. File .csr (Certificate Signing Request) giống như tờ đơn xin xác nhận của bạn, phải bạn cần mang đi xin ông CA đóng dấu cho. Và vì mình đã tự đóng vai trò CA (với 2 file .key và .pem đã tạo ở bước trước) thì mình sẽ tự đóng dấu cho yêu cầu này:

```
openssl x509 -req -days 3650 -in dongna_app.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out dongna_app.crt -extensions v3_req -extfile openssl.cnf
```

Kết quả sẽ sinh ra file dongna_app.crt. Bây giờ ta sẽ tạo file dongna_app.pem từ 3 file dongna_app.key, dongna_app.csr và dongna_app.crt
```
cat dongna_app.key > dongna_app.pem
cat dongna_app.csr >> dongna_app.pem
cat dongna_app.crt >> dongna_app.pem
```

# Import CA vừa tạo vào trình duyệt web(client)
Trước hết các bạn tải file rootCA.pem đã tạo ở trên về máy client.

Sau đó mở Chrome và vào địa chỉ này để vào mục setting: chrome://settings/security --> Chọn vào Manage certificates Trong hộp thoại hiện ra bạn vào tab Trusted Root Certification Authorities --> Import --> Next --> Browse --> Chọn file rootCA.pem --> Next --> Next --> Finish.
![image](https://github.com/user-attachments/assets/1ae34569-2817-4d4c-b366-164d72908fc6)

Như vậy là ta đã tự trở thành một CA và được 2 trình duyệt trên máy client tin tương như các CA "Auth" khác.



# Cấu hình Haproxy làm SSL Termination (SSL Offload)

Bước này đã được cấu hình ở [đây](https://github.com/nguyenanhdongvn/Document/blob/main/Document/F%20-%20Load%20Balancing%20tr%C3%AAn%20Kubernetes%20d%C3%B9ng%20Haproxy%20v%C3%A0%20Nginx-Ingress.md), tuy nhiên ta sẽ đi chi tiết vào config của haproxy 

```
frontend frontend_ssl_443
        bind :80
        bind *:443 ssl crt /etc/haproxy/ssl/dongna_app.pem
        mode http
        option httpclose
        option forwardfor
        reqadd X-Forwarded-Proto:\ https        
        cookie  SRVNAME insert indirect nocache
        default_backend backend_ingress        

        acl rancher hdr_dom(host) -i rancher.monitor.viettq.com
        use_backend backend_rancher if rancher
```


Trong đó:
- **frontend frontend_ssl_443**: Chỉ ra một frontend có tên là frontend_ssl_443
- **bind :80**: Chỉ ra frontend sẽ listen ở port 80
- **bind *:443 ssl crt /etc/haproxy/ssl/viettq_app.pem**:
  - bind *:443: Chỉ ra frontend sẽ listen ở port 443 ở tất cả các network interface (chú ý dấu * trước port 443)
  - SSL là bật tính năng SSL Termination cho listener này.
  - CRT chỉ ra đường dẫn tới file SSL-Certificate, ta đã thực hiện tạo ở bước trước cần copy lên máy chủ cài haproxy và cấu hình đường dẫn vào đây.
- **reqadd X-Forwarded-Proto:\ https**: Thêm https header và cuối HTTPS request
- **default_backend backend_ingress**: Cấu hình mặc định request nếu ko match với rule ALC nào thì sẽ vào backend là backend_ingress, đây là rule để mặc định sẽ kết nối tới các app trên K8S thông qua Nginx-Ingress
- **acl rancher hdr_dom(host) -i rancher.monitor.dongna.com**: Tạo điều kiện check rancher nếu host request tới trùng với địa chỉ "rancher.monitor.dongna.com"
- **use_backend backend_rancher if rancher**: Nếu điều kiện rancher là đúng thì trỏ tới backend là backend_rancher

Cấu hình backend_ingress: Thực hiện load balancing request tới 3 k8s master node, port 30080 là Node Port của Nginx-Ingress

```
backend backend_ingress
        mode    http
        stats   enable
        stats   auth username:password
        balance roundrobin
        server  master1 192.168.10.11:30080 cookie p1 weight 1 check inter 2000
        server  master2 192.168.10.12:30080 cookie p1 weight 1 check inter 2000
        server  master3 192.168.10.13:30080 cookie p1 weight 1 check inter 2000
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
192.168.10.10 rancher.monitor.dongna.com
```

Đồng thời phải khai ingress rule trên K8S để forward từ host apple.demo.viettq.com tới service apple-service như sau:
![image](https://github.com/user-attachments/assets/b0a6ac80-8cbd-4f76-b015-b9a79356972e)



