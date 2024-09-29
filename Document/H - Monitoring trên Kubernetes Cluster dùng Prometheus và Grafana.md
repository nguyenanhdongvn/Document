# Cài đặt Prometheus và Grafana trên K8S bằng Helm
* Tạo thư mục để chứa file cấu hình cài đặt Prometheus:
```
mkdir /home/sysadmin/kubernetes_installation/prometheus
```

* Khai báo repo của Helm và download helm-chart của Prometheus về:
```
cd /home/sysadmin/kubernetes_installation/prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm search repo prometheus |egrep "stack|CHART"
helm pull prometheus-community/kube-prometheus-stack --version 63.1.0
tar -xzf kube-prometheus-stack-63.1.0.tgz
cp kube-prometheus-stack/values.yaml values-prometheus.yaml
```

- Chỉnh sửa qua file values mặc định của bộ helm-chart này sẽ thấy nó đã mặc định cấu hình Rule cho các thành phần của K8S từ etcd, kube-api.. Và Alert Manager cũng dc cài mặc định trong bộ này luôn.
-  - Cấu hình tạo ingress rule cho Alert Manager để point vào URL http://alertmanager.dongna.com/
```
alertmanager:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - alertmanager.dongna.com
    paths:
      - /
```
-  - Cấu hình password login vào Grafana UI là `P@ssw0rdP@ssw0rd`
```
grafana
  adminPassword: P@ssw0rdP@ssw0rd
```
-  - Cấu hình tạo ingress rule cho Grafana để point vào URL http://grafana.dongna.com/
```
grafana:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - grafana.dongna.com
    paths:
      - /
```

-  - Cấu hình tạo ingress rule cho Prometheus để point vào URL http://prometheus.dongna.com/
```
prometheus:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - prometheus.dongna.com
    paths:
      - /
```

Ta sẽ expose các ứng dụng này qua Nginx-Ingress nên tất cả các cấu hình service sẽ để mặc định là ClusterIP.

Sau khi đã cấu hình xong file value-prometheus.yaml thì ta cài đặt
```
kubectl create ns monitor
helm install prometheus-grafana-stack -f values-prometheus.yaml kube-prometheus-stack -n monitor
```

Kiểm tra cài đặt
```
kubectl get all -n monitor
```
Output
```
NAME                                                               READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-grafana-stack-k-alertmanager-0         2/2     Running   0          85s
pod/prometheus-grafana-stack-5495cbd7dc-bct76                      3/3     Running   0          89s
pod/prometheus-grafana-stack-k-operator-57d5b9f856-lsj4k           1/1     Running   0          88s
pod/prometheus-grafana-stack-kube-state-metrics-6d99dc7bd4-p8gw5   1/1     Running   0          89s
pod/prometheus-grafana-stack-prometheus-node-exporter-6fx7x        1/1     Running   0          89s
pod/prometheus-grafana-stack-prometheus-node-exporter-jk5rk        1/1     Running   0          88s
pod/prometheus-grafana-stack-prometheus-node-exporter-km8s2        1/1     Running   0          89s
pod/prometheus-grafana-stack-prometheus-node-exporter-n7gq9        1/1     Running   0          89s
pod/prometheus-grafana-stack-prometheus-node-exporter-psmlj        1/1     Running   0          88s
pod/prometheus-grafana-stack-prometheus-node-exporter-wzlbk        1/1     Running   0          88s
pod/prometheus-prometheus-grafana-stack-k-prometheus-0             2/2     Running   0          85s

NAME                                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                               ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   85s
service/prometheus-grafana-stack                            ClusterIP   10.233.58.71    <none>        80/TCP                       89s
service/prometheus-grafana-stack-k-alertmanager             ClusterIP   10.233.39.60    <none>        9093/TCP,8080/TCP            89s
service/prometheus-grafana-stack-k-operator                 ClusterIP   10.233.43.109   <none>        443/TCP                      89s
service/prometheus-grafana-stack-k-prometheus               ClusterIP   10.233.42.143   <none>        9090/TCP,8080/TCP            89s
service/prometheus-grafana-stack-kube-state-metrics         ClusterIP   10.233.30.80    <none>        8080/TCP                     89s
service/prometheus-grafana-stack-prometheus-node-exporter   ClusterIP   10.233.43.211   <none>        9100/TCP                     89s
service/prometheus-operated                                 ClusterIP   None            <none>        9090/TCP                     85s

NAME                                                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/prometheus-grafana-stack-prometheus-node-exporter   6         6         6       6            6           kubernetes.io/os=linux   89s

NAME                                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-grafana-stack                      1/1     1            1           89s
deployment.apps/prometheus-grafana-stack-k-operator           1/1     1            1           89s
deployment.apps/prometheus-grafana-stack-kube-state-metrics   1/1     1            1           89s

NAME                                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-grafana-stack-5495cbd7dc                      1         1         1       89s
replicaset.apps/prometheus-grafana-stack-k-operator-57d5b9f856           1         1         1       88s
replicaset.apps/prometheus-grafana-stack-kube-state-metrics-6d99dc7bd4   1         1         1       89s

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-prometheus-grafana-stack-k-alertmanager   1/1     85s
statefulset.apps/prometheus-prometheus-grafana-stack-k-prometheus       1/1     85s
```

Kiểm tra ingress đã tạo
```
kubectl get ingress -n monitor
```

Output
```
NAME                                      CLASS   HOSTS                             ADDRESS   PORTS   AGE
prometheus-grafana-stack                  nginx   grafana.dongna.com                          80      109s
prometheus-grafana-stack-k-alertmanager   nginx   alertmanager.monitor.dongna.com             80      109s
prometheus-grafana-stack-k-prometheus     nginx   prometheus.dongna.com                       80      109s
```

Như vậy là đã cài xong Prometheus + Grafana + Alert Manager trên K8s Cluster. Ta thực hiện add host trên Local Host và thử truy cập vào web UI của Prometheus + Grafana + Alert Manager để kiểm tra<br>
C:\Windows\System32\drivers\etc
```
#monitoring
192.168.10.10 alertmanager.dongna.com
192.168.10.10 grafana.dongna.com
192.168.10.10 alertmanager.dongna.com
```

Ta có thể truy cập được Prometheus + Grafana + Alert Manager qua các URL
https://alertmanager.dongna.com/
https://grafana.dongna.com/
https://prometheus.dongna.com/

Prometheus + AlertManager + Grafana và các Node Exporter của K8s đã được cài đặt hoàn chỉnh. Ta sẽ dùng các metric lấy được từ Prometheus để hiển thị lên Grafana.<br>
Các dashboard trên grafana các bạn có thể dễ dàng tìm được các bộ template có sẵn tại [đây](https://grafana.com/grafana/dashboards/). Dashboard giám sát K8s thì có thể tìm google với từ khóa grafana k8s node dashboard sẽ ra một loạt các template cho bạn lựa chọn <br>
VD: Dashboard ID 315 <br>
![image](https://github.com/user-attachments/assets/6b4d4634-aed0-4c2b-998f-ee95056bc0ee)

Import dashboard này vào Grafana như sau: Create --> Import --> Nhập ID là 315 và ấn Load.

![image](https://github.com/user-attachments/assets/8d57e2b5-4a2d-44c8-9564-1c2450c39051)

# Giám sát application trên K8S bằng Prometheus và Grafana
Sau khi cài đặt Prometheus + Grafana thì Prometheus đã lấy được các metrics của K8s và hiển thị lên dashboard của Grafana. <br>
Vậy khi ta deploy một application khác lên K8s thì làm sao để lấy được metrics của application và hiển thị lên dashboard?
Ta sẽ thử deploy application Minio lên K8S và lấy metric của nó qua Prometheus rồi hiển thị lên dashboard của Grafana.<br>
Ý tưởng là cài Minio lên k8s (bạn nào chưa biết về Minio thì google thêm nhé, nó cũng khá phổ biến với ai hay dùng open-source). Sau đó cấu hình cho nó expose metric của nó, vì khá nhiều opensource hỗ trợ sẵn metric mà Prometheus có thể hiểu được.

# Cài đặt Minio
Cài đặt application Minio bằng helm chart
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo minio
helm pull bitnami/minio --version=14.7.13
tar -xzf minio-14.7.13.tgz
cp minio/values.yaml  values-minio.yaml
```

chỉnh sửa các tham số ở file value-minio.yaml
```
rootPassword: P@ssw0rdP@ssw0rd
ingress:
  enabled: true
  ingressClassName: nginx
  hostname: minio.dongna.com
  annotations:
    prometheus.io/scrape: 'true'
persistence:
  storageClass: longhorn-storage-retain
  size: 1Gi
```

Deploy application minio bằng helm lên 
```
kubectl create ns dongna-prod
helm install minio -f values-minio.yaml minio -n dongna-prod
```
Phần cấu hình LB ta đã cài đặt HAproxy và Keepalived từ trước, nên giờ chỉ cần add hostname ở Local Host là có thể truy cập <br>
C:\Windows\System32\drivers\etc
```
192.168.10.10 minio.dongna.com
```
![image](https://github.com/user-attachments/assets/faeb34ad-8212-4282-b86c-c98cd4c9f331)

Đăng nhập bằng user "admin" và password như đã cấu hình trong file value-minio.yaml. Tạo ra 3 bucket để check Dashboard trên Grafana có hiển thị chính xác không:

Trở lại với file value-minio.yaml, ta sẽ thấy có tham số paths: - /minio/v2/metrics/cluster, đây là endpoint để Minio expose metric của nó cho Prometheus pull về
```
metrics:
  serviceMonitor:
    paths:
      - /minio/v2/metrics/cluster
      - /minio/v2/metrics/node
```

# Cấu hình Prometheus pull metrics từ Minio
Trở lại với file value-prometheus.yaml, ta update thêm phần scrape-config cho Minio như sau
```
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: minio-job
        metrics_path: /minio/v2/metrics/cluster
        scheme: http
        static_configs:
          - targets: ['minio.dongna-prod.svc.cluster.local:9000']
```
Trong đó:
- **metric_path**: là endpoint mà minio dùng để expose metrics
- **targets**:  là IP/Port để expose metrics. Ở đây ta có service name là "minio" và port của Minio là "9000". Do Prometheus và Minio nằm ở 2 namespace khác nhau nên ta phải đặt service name ở dạng full: <service-name>.<namespace>.svc.<cluster-name> trong đó cluster name đang set là "cluster.local"

## Helm Upgrade Prometheus để cập nhật value mới vừa thay đổi:
```
helm upgrade prometheus-grafana-stack -f values-prometheus.yaml kube-prometheus-stack -n monitor
```

Kiểm tra trên prometheus.dongna.com xem đã có Metric của Minio hay chưa
![image](https://github.com/user-attachments/assets/54efd6a6-84e0-4989-926b-780dccfa163a)

Vậy là Prometheus đã lấy được metrics của Minio rồi. Tiếp theo là hiển thị lên dashboard của Grafana
Search template dashboard cho minio trên mạng cho tiện lợi. Keyword search là "grafana minio dashboard", ta tìm dc một dashboard ID là 13502 (tham khảo: https://grafana.com/grafana/dashboards/13502)
![image](https://github.com/user-attachments/assets/b7b84dc7-384d-48d2-97fc-00f79b2b1138)

Như vậy là các Metrics của Minio đã được hiển thị lên dashboard của Grafana. Có thể thấy Metric "Number of Bukets" đang là 3, đúng với số lượng bucket ta đã tạo

# Cấu hình alert qua Alert Manager
Để demo, ta cấu hình rule alert cho Minio với Metric là số lượng file của 1 bucket nếu lớn hơn 1 thì sẽ có alert

Tiếp tục sửa value-prometheus.yaml, thêm rule cảnh báo như sau:
```
defaultRules
  additionalPrometheusRules:
  - name: minio-rule-file
    groups:
    - name: minio-rule
      rules:
      - alert: MinioBucketUsage
        expr: minio_bucket_usage_object_total{bucket="dong", instance="minio.dongna-prod.svc.cluster.local:9000", job="minio-job"} > 1
        for: 10s
        labels:
          severity: page
        annotations:
          summary: High bucket usage
```

Trong đó:
- Rule trên thực hiện theo dõi bucket "dong" nếu có số lượng object > 1 trong 10s thì sẽ có alert.

Helm upgrade để cập nhật các thay đổi của prometheus release
```
helm upgrade prometheus-grafana-stack -f values-prometheus.yaml kube-prometheus-stack -n monitor
```





