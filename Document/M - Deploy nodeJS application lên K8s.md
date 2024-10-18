# Giới thiệu
- Sau khi đã hoàn thành cài đặt xong những thành phần thiết yếu như:
  - K8s cluster
  - Storage Class (Longhorn và NFS)
  - LoadBalance (Haproxy và nginx-ingress)
  - Private Registry (Harbor)

- Giờ ta đã có thể deploy 1 application lên K8s, ta sẽ thử deploy 1 nodeJS application với các bước như sau:
  - Cài đặt environment để deploy nodeJS application
  - Code 1 module NodeJS nhỏ và trả về web html
  - Build Docker Image cho application
  - Deploy application lên K8s bằng các file manifest (.yaml)

# Cài đặt environment để build NodeJS application
- Ta cài môi trường lên `cicd` server để build ứng dụng ở đây và triển khai lên k8s.
```
sudo yum clean all && sudo yum makecache
sudo yum install -y gcc-c++ make
sudo yum install -y nodejs
sudo yum install -y npm
```
- Pull source code về
```
git clone https://github.com/rockman88v/nodejs-demo-k8s.git
```

**NOTE: Các file html ở đây là các template trên mạng và đã được customize một chút. Code nodejs cũng là lấy ví dụ trên mạng và sửa lại để demo cho việc sử dụng các biến môi trường từ K8s vào ứng dụng nodejs.**

- Cơ bản app nó làm nhiệm vụ sau:
  - Gọi vào "/" trả về file index.html, trong đó có cập nhật giá trị các biến môi trường liên quan tới triển khai (POD_NAME, POD_IP..)
  - Gọi vào "/about" hay "/about-us" trả về about.html
  - Các request khác thì trả về 404.html
Ở đây đã có sẵn node_modules rồi, hoặc các bạn có thể cài lại bằng lệnh:
```
npm install
```

# Build Docker Image và push lên Private Registry
- Ta đã có Harbor là Private Registry (harbor.dongna.com),
```
docker build -t harbor.dongna.com/dongna_project/demo-app:v1 .
```

- Push lên 
```
docker push harbor.dongna.com/dongna_project/demo-app:v1
```

Deploy application lên K8s
- Để triển khai lên k8s thì ta sẽ dùng 3 resource chính là Deployment, Service và Ingress.
- Trong cấu hình Deployment có gán các biến môi trường lấy từ thông tin metadata của Pod khi deploy lên K8s
- Service cài đặt nodeport 31123 để kết nối trực tiếp qua IP của Node
- Ingress cài đặt ở địa chỉ host là demo-app.dongna.com


