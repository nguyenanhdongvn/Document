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
- Build Image
```
cd ~/nodejs-demo-k8s
docker build -t harbor.dongna.com/demo-nodejs/demo-nodejs:v1 .
```
- Output
```
[+] Building 1.1s (10/10) FINISHED                                                                                                                                                             docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                     0.0s
 => => transferring dockerfile: 153B                                                                                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/node:10                                                                                                                                               0.9s
 => [internal] load .dockerignore                                                                                                                                                                        0.0s
 => => transferring context: 2B                                                                                                                                                                          0.0s
 => [1/5] FROM docker.io/library/node:10@sha256:59531d2835edd5161c8f9512f9e095b1836f7a1fcb0ab73e005ec46047384911                                                                                         0.0s
 => [internal] load build context                                                                                                                                                                        0.0s
 => => transferring context: 31.88kB                                                                                                                                                                     0.0s
 => CACHED [2/5] WORKDIR /usr/src/app                                                                                                                                                                    0.0s
 => CACHED [3/5] COPY package*.json ./                                                                                                                                                                   0.0s
 => CACHED [4/5] RUN npm install                                                                                                                                                                         0.0s
 => CACHED [5/5] COPY . .                                                                                                                                                                                0.0s
 => exporting to image                                                                                                                                                                                   0.0s
 => => exporting layers                                                                                                                                                                                  0.0s
 => => writing image sha256:a0f9c226a055b64138c2165216bcc21d6da8f2d9caa37cb0289122672a0c0de5                                                                                                             0.0s
 => => naming to harbor.dongna.com/demo-nodejs/demo-nodejs:v1  
```

- Push Image lên Harbor Registry
```
docker push harbor.dongna.com/demo-nodejs/demo-nodejs:v1
```
- Output
```
The push refers to repository [harbor.dongna.com/demo-nodejs/demo-nodejs]
2ee2ff9ba892: Pushed
c5efb49a5aee: Pushed
eaaa5a4d1436: Pushed
34ece8f00e48: Pushed
3ab01e8988bf: Pushed
c98dc9a94132: Pushed
3ffdb7e28503: Pushed
33dd93485756: Pushed
607d71c12b77: Pushed
052174538f53: Pushed
8abfe7e7c816: Pushed
c8b886062a47: Pushed
16fc2e3ca032: Pushed
v1: digest: sha256:5c103e0445f9319e7a3056d52f1e03183240b92b24dbf7e8a13c96091c23fea2 size: 3052
```
# Deploy application lên K8s
- Để triển khai lên k8s thì ta sẽ dùng 3 resource chính là `Deployment`, `Service` và `Ingress`.
- Trong cấu hình Deployment có gán các biến môi trường lấy từ thông tin metadata của Pod khi deploy lên K8s
- Service cài đặt nodeport 31123 để kết nối trực tiếp qua IP của Node
- Ingress cài đặt ở địa chỉ host là demo-nodejs.dongna.com

- Trên `local machine`, tạo folder chứa manifest file
```
mkdir -p $HOME/k8s/manifest/
mkdir -p $HOME/k8s/manifest/demo-nodejs && cd $HOME/k8s/manifest/demo-nodejs
```

- Tạo manifest file `deployment.yaml`
```
cat <<EOF > $HOME/k8s/manifest/demo-nodejs/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-nodejs-deployment
  labels:
    app: demo-nodejs-dongna
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-nodejs-dongna
  template:
    metadata:
      labels:
        app: demo-nodejs-dongna
    spec:
      containers:
        - name: demo-nodejs
          image: harbor.dongna.com/demo-nodejs/demo-nodejs:v1
          imagePullPolicy: Always
          resources:
            # Specifying the resourses that we might need for our application
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
            - containerPort: 8080
          env:
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: MY_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: MY_POD_SERVICE_ACCOUNT
              valueFrom:
                fieldRef:
                  fieldPath: spec.serviceAccountName
EOF
```

- Tạo manifest file `service.yaml`
```
cat <<EOF > $HOME/k8s/manifest/demo-nodejs/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-nodejs-service
spec:
  selector:
    app: demo-nodejs-dongna
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 8080
      nodePort: 31123
EOF
```

- Tạo manifest file `ingress.yaml`
```
cat <<EOF > $HOME/k8s/manifest/demo-nodejs/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-nodejs-ingress
  labels:
    name: demo-nodejs-dongna
spec:
  ingressClassName: nginx
  rules:
  - host: demo-nodejs.dongna.com
    http:
      paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: demo-nodejs-service
              port:
                number: 5000
EOF
```

- Deploy application vào namespace `demo-nodejs`
```
kubectl create ns demo-nodejs
kubectl -n demo-nodejs apply -f $HOME/k8s/manifest/demo-nodejs/deployment.yaml
kubectl -n demo-nodejs apply -f $HOME/k8s/manifest/demo-nodejs/service.yaml
kubectl -n demo-nodejs apply -f $HOME/k8s/manifest/demo-nodejs/ingress.yaml
```

**NOTE: Ta sẽ gặp phải error là POD không thể pull image từ Harbor Registry được vì trên Worker Node không resolve được hostname harbor.dongna.com và vì containerd không có config để trỏ về Private Registry là Harbor**
```
  Normal   Scheduled  108s                default-scheduler  Successfully assigned demo/nodejs-app-deployment-bbc456648-ccwhp to worker3
  Normal   BackOff    33s (x2 over 84s)   kubelet            Back-off pulling image "harbor.dongna.com/demo/demo-app:v1"
  Warning  Failed     33s (x2 over 84s)   kubelet            Error: ImagePullBackOff
  Normal   Pulling    22s (x3 over 107s)  kubelet            Pulling image "harbor.dongna.com/demo/demo-app:v1"
  Warning  Failed     1s (x3 over 85s)    kubelet            Failed to pull image "harbor.dongna.com/demo/demo-app:v1": failed to pull and unpack image "harbor.dongna.com/demo/demo-app:v1": failed to resolve reference "harbor.dongna.com/demo/demo-app:v1": failed to do request: Head "https://harbor.dongna.com/v2/demo/demo-app/manifests/v1": dial tcp 125.235.4.59:443: connect: connection refused
  Warning  Failed     1s (x3 over 85s)    kubelet            Error: ErrImagePull
```

**Để giải quyết vấn đề này ta sẽ add host và thêm config cho containerd trỏ về Harbor Registry trên các Worker Node**
```
sudo su -

# Add host record for K8s Node to resolve harbor.dongna.com
cat << EOF >> /etc/hosts

# Harbor Registry
192.168.10.20 harbor.dongna.com
EOF

# Setup Harbor Registry for connecting from K8s
cat <<EOF >> /etc/containerd/config.toml
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor.dongna.com"]
           endpoint = ["https://harbor.dongna.com"]
      [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.dongna.com".tls]
           insecure_skip_verify = true
      [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.dongna.com".auth]
           username = "dongna"
           password = "Life'sg00d123qwe"
EOF

# Restart containerd for applying Harbor Registry config
systemctl restart containerd
```

- Kiểm tra
```
dong@DONG-PC:~/k8s/manifest/demo-nodejs$ k get all -n demo-nodejs
NAME                                          READY   STATUS    RESTARTS   AGE
pod/demo-nodejs-deployment-684f6fc5db-dlkrn   1/1     Running   0          16m
pod/demo-nodejs-deployment-684f6fc5db-dllcz   1/1     Running   0          10m
pod/demo-nodejs-deployment-684f6fc5db-gkqr6   1/1     Running   0          8m15s

NAME                          TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/demo-nodejs-service   LoadBalancer   10.233.9.97   <pending>     5000:31123/TCP   16m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demo-nodejs-deployment   3/3     3            3           16m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/demo-nodejs-deployment-684f6fc5db   3         3         3       16m


dong@DONG-PC:~/k8s/manifest/demo-nodejs$ k get ingress -n demo-nodejs
NAME                  CLASS   HOSTS                    ADDRESS   PORTS   AGE
demo-nodejs-ingress   nginx   demo-nodejs.dongna.com             80      17m
```

- Như vậy là mọi thứ đã hoàn thành, ta truy cập thử vào https://demo-nodejs.dongna.com để xem kết quả <br>
![image](https://github.com/user-attachments/assets/4d79f097-5d31-49c9-899d-9c345da0c1c9) <br>
![image](https://github.com/user-attachments/assets/4fae14f7-fd72-4aa1-86fd-836628391f34) <br>
![image](https://github.com/user-attachments/assets/da1d1c57-3b21-4add-97a0-f30a605eedb9) <br>

- Refresh web page thử thì ta thấy nodejs app được random chuyển sang pod khác
