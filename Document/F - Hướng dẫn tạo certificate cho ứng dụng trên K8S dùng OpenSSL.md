# Tạo Certificate Authority trên "cicd" node
Để có Certificate xịn thì bạn phải mua của các CA. Ta cũng có thể tự tạo một CA ở trên Local và import vào trình duyệt để trình duyệt hiểu là CA (mà chúng ta tạo ra) là một nhà cung cấp CA tin cậy. Sau đó, các Certificate được cấp bởi CA này sẽ được coi là validate.

## Tạo private key cho CA (bắt buộc phải nhập pass phrase)
Trên `cicd` server tạo private key cho CA tại /home/dong/ssl/
```
mkdir /home/dong/ssl && cd /home/dong/ssl
openssl genrsa -des3 -out rootCA.key 2048
```

## Tạo file pem từ file private key (nhập pass của rootCA đã tạo bên trên)
```
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1825 -out rootCA.pem
```

Output
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
sysadmin@cicd:~$ ll /home/dong/ssl/
total 8
-rw-------. 1 sysadmin sysadmin 1862 Sep 23 22:33 rootCA.key
-rw-r--r--. 1 sysadmin sysadmin 1354 Sep 23 22:35 rootCA.pem
```
# Tạo SSL Certificate cho ứng dụng web ở local
Đầu tiên ta tạo một file openssl.cnf để cấu hình thêm thông tin SAN như sau:
```
cat <<EOF > /home/dong/ssl/openssl.cnf
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

Rồi tới bước quan trọng nhất là mang CSR đi đăng ký. File .csr (Certificate Signing Request) giống như tờ đơn xin xác nhận của bạn, phải bạn cần mang đi xin ông CA đóng dấu cho. Và vì mình đã tự đóng vai trò CA (với 2 file .key và .pem đã tạo ở bước trước) thì mình sẽ tự đóng dấu cho yêu cầu này:

```
openssl x509 -req -days 3650 -in dongna_app.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out dongna_app.crt -extensions v3_req -extfile openssl.cnf
```

Kết quả sẽ sinh ra file dongna_app.crt. Bây giờ ta sẽ tạo file dongna_app.pem từ 3 file dongna_app.key, dongna_app.csr và dongna_app.crt
```
cat dongna_app.key > dongna_app.pem
cat dongna_app.csr >> dongna_app.pem
cat dongna_app.crt >> dongna_app.pem
```

Kiểm tra lại các file cert xem đã tạo đủ chưa

```
sysadmin@cicd:~$ ll /home/dong/ssl/
total 36
-rw-r--r-- 1 sysadmin sysadmin 1403 Sep 29 01:58 dongna_app.crt
-rw-r--r-- 1 sysadmin sysadmin 1102 Sep 29 00:56 dongna_app.csr
-rw------- 1 sysadmin sysadmin 1704 Sep 29 00:55 dongna_app.key
-rw-r--r-- 1 sysadmin sysadmin 4209 Sep 29 01:59 dongna_app.pem
-rw-r--r-- 1 sysadmin sysadmin  645 Sep 29 00:54 openssl.cnf
-rw------- 1 sysadmin sysadmin 1862 Sep 29 00:50 rootCA.key
-rw-r--r-- 1 sysadmin sysadmin 1354 Sep 29 00:52 rootCA.pem
-rw-r--r-- 1 sysadmin sysadmin   41 Sep 29 01:58 rootCA.srl
```


# Import CA vừa tạo vào trình duyệt web(client)
Trước hết các bạn tải file rootCA.pem đã tạo ở trên về máy client.

Sau đó mở Chrome và vào địa chỉ này để vào mục setting: chrome://settings/security --> Chọn vào Manage certificates Trong hộp thoại hiện ra bạn vào tab Trusted Root Certification Authorities --> Import --> Next --> Browse --> Chọn file rootCA.pem --> Next --> Next --> Finish.
![image](https://github.com/user-attachments/assets/1ae34569-2817-4d4c-b366-164d72908fc6)

Khi warning box "Security Warning" hiện lên bạn chọn vào Yes.
![image](https://github.com/user-attachments/assets/4b739381-4675-4cb5-b017-fdccdf5cff62)

Lúc này thông tin CA đã được import vào Google Chrome.
![image](https://github.com/user-attachments/assets/c7ea91cf-62c5-4a75-9e98-5b2b98be117c)

Như vậy là ta đã tự trở thành một CA và được trình duyệt Google Chrome trên máy user (local) trust như các CA "Auth" khác.
