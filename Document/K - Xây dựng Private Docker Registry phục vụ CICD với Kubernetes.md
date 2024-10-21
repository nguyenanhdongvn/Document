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
## Tạo folder chứa cert cho Harbor Registry
```
mkdir -p /home/sysadmin/harbor-registry/certs
cp /home/sysadmin/ssl/dongna_app.crt /home/sysadmin/harbor-registry/certs/
cp /home/sysadmin/ssl/dongna_app.key /home/sysadmin/harbor-registry/certs/
```
## Tạo folder cài đặt và folder chứa data của Harbor Registry
```
mkdir -p /data/harbor-data
mkdir -p /home/sysadmin/harbor-registry
cd /home/sysadmin/harbor-registry
curl -s https://api.github.com/repos/goharbor/harbor/releases/latest | grep browser_download_url | cut -d '"' -f 4 | grep '\.tgz$' | wget -i -
tar xvzf harbor-offline-installer*.tgz
cd /home/sysadmin/harbor-registry/harbor
cp harbor.yml.tmpl harbor.yml
```
## Cấu hình file value.yaml của Harbor Registry helm chart
```
hostname: harbor.dongna.com
certificate: /home/sysadmin/harbor-registry/certs/dongna_app.crt
private_key: /home/sysadmin/harbor-registry/certs/dongna_app.key
harbor_admin_password: P@ssw0rdP@ssw0rd
password: P@ssw0rdP@ssw0rd
data_volume: /data/harbor-data
```

# Cài đặt Harbor Registry
- Cài đặt Harbor Registry bằng lệnh
```
cd /home/sysadmin/harbor-registry/harbor
./install.sh
```

Output
```
[Step 0]: checking if docker is installed ...

Note: docker version: 27.3.1

[Step 1]: checking docker-compose is installed ...

Note: Docker Compose version v2.29.7

[Step 2]: loading Harbor images ...
Loaded image: goharbor/prepare:v2.11.1
Loaded image: goharbor/harbor-db:v2.11.1
Loaded image: goharbor/harbor-exporter:v2.11.1
Loaded image: goharbor/redis-photon:v2.11.1
Loaded image: goharbor/nginx-photon:v2.11.1
Loaded image: goharbor/harbor-portal:v2.11.1
Loaded image: goharbor/harbor-core:v2.11.1
Loaded image: goharbor/harbor-log:v2.11.1
Loaded image: goharbor/harbor-jobservice:v2.11.1
Loaded image: goharbor/harbor-registryctl:v2.11.1
Loaded image: goharbor/registry-photon:v2.11.1
Loaded image: goharbor/trivy-adapter-photon:v2.11.1


[Step 3]: preparing environment ...

[Step 4]: preparing harbor configs ...
prepare base dir is set to /home/sysadmin/open-sources/harbor_registry/harbor
Generated configuration file: /config/portal/nginx.conf
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /data/secret/keys/secretkey
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir


Note: stopping existing Harbor instance ...
WARN[0000] /home/sysadmin/open-sources/harbor_registry/harbor/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion


[Step 5]: starting Harbor ...
WARN[0000] /home/sysadmin/open-sources/harbor_registry/harbor/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
[+] Running 10/10
 ✔ Network harbor_harbor        Created                                                                                                                                                                  0.2s
 ✔ Container harbor-log         Started                                                                                                                                                                  0.6s
 ✔ Container registry           Started                                                                                                                                                                  1.5s
 ✔ Container harbor-db          Started                                                                                                                                                                  2.0s
 ✔ Container harbor-portal      Started                                                                                                                                                                  1.8s
 ✔ Container redis              Started                                                                                                                                                                  1.9s
 ✔ Container registryctl        Started                                                                                                                                                                  2.1s
 ✔ Container harbor-core        Started                                                                                                                                                                  2.5s
 ✔ Container nginx              Started                                                                                                                                                                  3.5s
 ✔ Container harbor-jobservice  Started                                                                                                                                                                  3.3s
✔ ----Harbor has been installed and started successfully.----
```

- Kiểm tra lại xem Harbor Registry đã chạy chưa
```
docker compose ps

NAME                IMAGE                                 COMMAND                  SERVICE       CREATED          STATUS                    PORTS
harbor-core         goharbor/harbor-core:v2.11.1          "/harbor/entrypoint.…"   core          12 minutes ago   Up 12 minutes (healthy)
harbor-db           goharbor/harbor-db:v2.11.1            "/docker-entrypoint.…"   postgresql    12 minutes ago   Up 12 minutes (healthy)
harbor-jobservice   goharbor/harbor-jobservice:v2.11.1    "/harbor/entrypoint.…"   jobservice    12 minutes ago   Up 12 minutes (healthy)
harbor-log          goharbor/harbor-log:v2.11.1           "/bin/sh -c /usr/loc…"   log           12 minutes ago   Up 12 minutes (healthy)   127.0.0.1:1514->10514/tcp
harbor-portal       goharbor/harbor-portal:v2.11.1        "nginx -g 'daemon of…"   portal        12 minutes ago   Up 12 minutes (healthy)
nginx               goharbor/nginx-photon:v2.11.1         "nginx -g 'daemon of…"   proxy         12 minutes ago   Up 12 minutes (healthy)   0.0.0.0:80->8080/tcp, [::]:80->8080/tcp, 0.0.0.0:443->8443/tcp, [::]:443->8443/tcp
redis               goharbor/redis-photon:v2.11.1         "redis-server /etc/r…"   redis         12 minutes ago   Up 12 minutes (healthy)
registry            goharbor/registry-photon:v2.11.1      "/home/harbor/entryp…"   registry      12 minutes ago   Up 12 minutes (healthy)
registryctl         goharbor/harbor-registryctl:v2.11.1   "/home/harbor/start.…"   registryctl   12 minutes ago   Up 12 minutes (healthy)
```

## Truy cập vào Harbor Registry UI:
- Cấu hình file host trên local host như sau:<br>
C:\Windows\System32\drivers\etc
```
192.168.10.20 harbor.dongna.com
```

- Truy cập vào url https://harbor.dongna.com/, đăng nhập bằng user `admin` và password đã cấu hình trong value.yaml trước đó
![image](https://github.com/user-attachments/assets/f2632e12-cfa2-423b-8a62-33351161fb23)

## Tạo Project, User và phân quyền User
- User > New User -> nhập thông tin User (dongna/P@ssw0rdP@ssw0rd)
![image](https://github.com/user-attachments/assets/6e2d491d-d1fd-457d-a97d-d46ed4685546)

- Gán quyền cho user `dongna` vào project `dongna_project`
![image](https://github.com/user-attachments/assets/29c9701f-9740-4b00-bf41-d3718d56e756)


## Cấu hình kết nối cho Developer (local host) để pull/push/ image lên Harbor Registry
- Trên local host, tạo folder chứa cert của Harbor Registry
```
sudo su -
mkdir -p /etc/docker/certs.d/harbor.dongna.com/
cd /etc/docker/certs.d/harbor.dongna.com/
```

- Từ local host, login vào Harbor Registry
```
docker login harbor.dongna.com
```

- Output
```
Username: dongna
Password:
WARNING! Your password will be stored unencrypted in /home/dong/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credential-stores

Login Succeeded
```

- Từ local host, ta thử pull image `hello-world` từ dockerhub về rồi đánh tag mới cho image này và push image này lên lại Harbor Registry
```
docker pull hello-world
docker tag hello-world:latest harbor.dongna.com/demo/hello-world:latest
docker push harbor.dongna.com/demo/hello-world:latest
```

- Output
```
Using default tag: latest
The push refers to repository [harbor.dongna.com/dongna_project/hello-world]
ac28800ec8bb: Pushed
latest: digest: sha256:d37ada95d47ad12224c205a938129df7a3e52345828b4fa27b03a98825d1e2e7 size: 524
```

- Như vậy là tao đã push thành công image lên Harbor Registry

