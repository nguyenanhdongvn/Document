# Giới thiệu
- Để deploy application lên K8s cần tới Docker Registry để lưu trữ Docker Image cho project. Tùy vào từng công ty mà sử dụng các giải pháp Registry khác nhau. 
- Với Cloud-based System, ta có thể sử dụng Docker Registry của Cloud (`AWS ECR`, `Azure ACR`, `Google GCR`).
- Với On-premise System, ta phải tự dựng 1 Registry riêng (Gitlab Registry, Private Docker Registry, Harbor Registry)

- Thông thường, ta có server đã cài docker, thì ta có thể docker pull để download docker image.
```
docker pull <image-name>:version
```
- Khi đó docker client sẽ kết nối tới Docker Hub (default) để tìm kiếm và pull docker image về server. Tuy nhiên khi ta triển khai một project yêu cầu phải có private Registry, ta cần phải dựng một private Registry riêng.
- Private Registry giúp ta quản lý và chia sẻ Docker Image với member trong team một cách nội bộ mà không cần phải ra ngoài Internet
- Việc này cũng giúp tăng tốc độ deploy do băng thông của Internal nhanh hơn so với băng thông Internet. Cũng như việc công ty sẽ không muốn lưu Docker Image lên các nền tảng public Registry.
![image](https://github.com/user-attachments/assets/a8ca3b99-c28c-4e8a-96d4-38650afca7ab)

# Harbor Registry
- Ta sẽ cài Harbor Registry và sử dụng domain name là harbor.dongna.com
- Tạo folder chứa cert cho Harbor Registry
```
mkdir -p /home/sysadmin/open-sources/harbor_registry/certs
cp /home/sysadmin/ssl/dongna_app.crt /home/sysadmin/open-sources/harbor_registry/certs/
cp /home/sysadmin/ssl/dongna_app.key /home/sysadmin/open-sources/harbor_registry/certs/
```
- Tạo folder cài đặt và folder chứa data của habor
```
sudo mkdir -p /data/harbor_data
mkdir -p /home/sysadmin/open-sources/harbor_registry
cd /home/sysadmin/open-sources/harbor_registry
curl -s https://api.github.com/repos/goharbor/harbor/releases/latest | grep browser_download_url | cut -d '"' -f 4 | grep '\.tgz$' | wget -i -
tar xvzf harbor-offline-installer*.tgz
cd harbor
cp harbor.yml.tmpl harbor.yml
```



