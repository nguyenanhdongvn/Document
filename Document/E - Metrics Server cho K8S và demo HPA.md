# Giới thiệu
Phần này thì cơ bản là đơn giản, khi cài đặt lần đầu có thể bạn sẽ gặp phải một số vấn nho nhỏ phát sinh mà nếu chưa có kinh nghiệm debug trên K8S thì có thể sẽ bị stuck luôn thì hơi chán.
Bản thân mình lần đầu cài cũng phải troubleshoot toát mồ hôi, nên giờ sẽ note cho các bạn tiết kiệm thời gian nhất

# Cài đặt Metrics Server bằng helm-chart
Vẫn như mọi khi, mình sẽ đứng từ server `cicd` để bắt đầu cài đặt. Trước hết là tạo thư mục lưu file cài đặt:
```
mkdir /home/sysadmin/kubernetes_installation/metric-server/
```

Add repo và download helm chart của Metrics Server về folder đã tạo:
```
cd /home/sysadmin/kubernetes_installation/metric-server/
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm search repo metrics-server
helm pull metrics-server/metrics-server --version 3.12.1
tar -xzf metrics-server-3.12.1.tgz
helm install metric-server metrics-server -n kube-system
```

Tới đây cần edit lại deployment của Metrics Server để fix lỗi liên quan tới kết nối. Bạn chạy lệnh sau:
```
kubectl -n kube-system edit deploy metric-server-metrics-server
```

Sau đó thêm vào tham số dưới session "args" như sau:
```
...
    spec:
      containers:
      - args:
        ...
        - --kubelet-insecure-tls
        image: k8s.gcr.io/metrics-server/metrics-server:v0.6.1
        imagePullPolicy: IfNotPresent
...
```
Lưu lại và chờ Pod được update lại. Các bạn có thể tham khảo thêm về issue này tại [đây](https://github.com/kubernetes-sigs/metrics-server/issues/278)

Output
```
kubectl -n kube-system get pods |grep metric
metric-server-metrics-server-67d444b447-llgnc   1/1     Running   0          57s
```

Sau khi cài metric server cho k8s ta có thể check được load của các Node trong Cluster
```
kubectl top nodes
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
master1   178m         9%     2180Mi          69%
master2   166m         9%     2173Mi          68%
master3   169m         9%     1685Mi          53%
worker1   141m         7%     1707Mi          50%
worker2   144m         7%     1857Mi          54%
worker3   72m          3%     535Mi           15%
```

# Cấu hình Auto Scaling với HPA
Tạo folder chứa config của HPA
```
mkdir /home/sysadmin/kubernetes_installation/hpa-example
cd /home/sysadmin/kubernetes_installation/hpa-example
```
Ý tưởng của bài lab HPA này là bạn tạo ra một Deployment gồm 1 Pod chạy Apache và một service trỏ tới Pod đó. Sau đó bạn cấu hình HPA cho Deployment đó với điều kiện tải CPU của pod > 50% thì sẽ scale số pod lên, và khi tải giảm xuống thì số pod cũng dc giảm theo.

Tạo yaml file cho Deployment và Service

```
cat <<EOF >> /home/sysadmin/kubernetes_installation/hpa-example/php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
EOF
```

Tạo Deployment và Service từ file yaml trên 
```
kubectl apply -f /home/sysadmin/kubernetes_installation/hpa-example/php-apache.yaml
```

Output
```
kubectl get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/php-apache-d4cf67d68-hkpwz   1/1     Running   0          62s
pod/pod-longhorn-delete          1/1     Running   0          34h
pod/pod-longhorn-retain          1/1     Running   0          34h

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.233.0.1     <none>        443/TCP   46h
service/php-apache   ClusterIP   10.233.41.99   <none>        80/TCP    62s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/php-apache   1/1     1            1           62s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/php-apache-d4cf67d68   1         1         1       62s
```

Như vậy là các bạn có thể thấy đang có 1 Pod của php-apache đang chạy. Giờ ta sẽ cấu hình HPA cho cái Deployment này, để khi các Pod của nó tăng tải thì sẽ được auto scale:

```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

Output
```
kubectl get hpa
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   <unknown>/50%   1         10        0          8s
```

Giờ ta sẽ tăng load cho POD php-apache bằng cách chạy một POD gọi liên tục tới POD php-apache. Bạn nên mở 2 cửa sổ terminal, một cái chạy POD để tạo load, một cái để theo dõi trạng thái của Deployment để xem nó auto scale như thế nào.

```
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

Theo dõi quá trình HPA auto scale POD của Deployment
```
kubectl get hpa php-apache --watch
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          3m13s
php-apache   Deployment/php-apache   147%/50%   1         10        1          3m35s
php-apache   Deployment/php-apache   214%/50%   1         10        3          3m51s
php-apache   Deployment/php-apache   169%/50%   1         10        5          4m6s
php-apache   Deployment/php-apache   85%/50%    1         10        7          4m22s
php-apache   Deployment/php-apache   72%/50%    1         10        7          4m37s
php-apache   Deployment/php-apache   71%/50%    1         10        7          4m52s
```

Sau đó ta tắt POD generate load đi, CPU ko còn bị high load nữa (CPU < 50%) thì K8s sẽ auto scale down số POD của Deployment xuống 1 POD như ban đầu
```
kubectl get hpa php-apache --watch
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        7          21m
php-apache   Deployment/php-apache   0%/50%    1         10        7          24m
php-apache   Deployment/php-apache   0%/50%    1         10        1          24m
```

Trên đây là một ví dụ đơn giản để các bạn hiểu về ý tưởng hoạt động của Autoscale. Ngoài đặt ngưỡng RAM/CPU bạn có thể customize theo rất nhiều điều kiện khác tùy vào nhu cầu thực tế. Các bạn có thể tham khảo thêm về HPA trên trang của Kubernetes: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/




