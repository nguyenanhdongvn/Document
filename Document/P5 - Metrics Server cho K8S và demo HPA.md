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
helm pull metrics-server/metrics-server --version 3.8.2
tar -xzf metrics-server-3.8.2.tgz
helm install metric-server metrics-server -n kube-system
```

Tới đây cần edit lại deployment của Metrics Server để fix lỗi liên quan tới kết nối. Bạn chạy lệnh sau:
```
kubectl -n kube-system edit deploy metric-server-metrics-server
```

Sau đó sửa các tham số dưới session "args" như sau:
```
...
    spec:
      containers:
      - args:
        - --secure-port=4443
        - --cert-dir=/tmp
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-insecure-tls=true
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: k8s.gcr.io/metrics-server/metrics-server:v0.6.1
        imagePullPolicy: IfNotPresent
...
```
Lưu lại và chờ Pod được update lại. Các bạn có thể tham khảo thêm về issue này ở topic trên Github và StackOverFlow:<br>
https://github.com/kubernetes-sigs/metrics-server/issues/278<br>
https://stackoverflow.com/questions/68648198/metrics-service-in-kubernetes-not-working

Output
```
kubectl -n kube-system get pods |grep metric

```

