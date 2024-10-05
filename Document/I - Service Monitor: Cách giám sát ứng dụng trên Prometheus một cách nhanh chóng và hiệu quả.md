# Giới thiệu
## Mục tiêu
- Lấy Metrics của application bằng cách cấu hình Job trong Scrape Config
- Lấy Metrics của application bằng cách cấu hình Service Monitor
- Cài đặt một application hỗ trợ cả Metric và Service Monitor
- Troubleshoot các issue trong khi cấu hình

## Kiến trúc
![image](https://github.com/user-attachments/assets/072f413f-af2d-4397-b9d2-05b9cabd445a)

## Monitoring Workflow của Prometheus sẽ được mô tả như sau:
- Prometheus pull Metrics từ các Target cần được monitor, các Target này có thể chia thành 2 loại:
    - Target hỗ trợ expose Metrics (nghĩa là các Target cần được monitor đã có sẵn API để Prometheus có thể trỏ đến và pull Metrics)
    - Target không hỗ trợ expose Metrics (phải cài thêm Exporter cho Target để Exporter push Metrics về Prometheus)
    - **Note: Quá trình Prometheus pull Metrics từ Target được gọi là "Job". Job chứa thông tin Target mà nó cần pull Metrics cũng như thời gian lấy Metrics (Interval, Retention...). Và có 2 cách tạo Job** <br>
        - Cấu hình "Scrape Config" trong Prometheus config: Đây là cách thủ công, nếu số lượng Job lớn thì file config của Prometheus sẽ rất dài và rối, khó quản lý. Hơn nữa, mỗi lần update Scrape Config mới thì ta sẽ phải update lại Prometheus helm chart (helm upgrade)
        - Cấu hình "Server Monitor": Đây là cách hiệu quả hơn, ta sẽ quản lý từng Target với từng yaml file tương ứng. Hơn nữa, mỗi lần có Service Monitor object mới, ta không cần phải update lại Prometheus helm chart (ko cần helm upgrade)
- Khi Prometheus đã pull Metrics về thì nó sẽ enrich Metrics bằng cách gán thêm các Label như: namespace, jobname, servicename, ... rồi ghi vào Time-Series Database của Prometheus
- Prometheus load các Alert Rule (các hàm so sánh giá trị Metrics với các Threshold) để quyết định những Alert Rule nào cần được raise Alert để đẩy về Alert Manager
- Alert Manager sẽ có config riêng để route Notification đến các Receiver khác nhau. Alert Manager hỗ trợ khá nhiều Receiver như: Email, Slack, MsTeams, Telegram, ...

# Prometheus Job và Service Monitor
## Prometheus Job
- Job là nơi cấu hình cách lấy Metrics của Target (Applications, Nodes), Cấu hình của Job được đặt trong Scrape Config của Prometheus.
- Nếu không cấu hình Job trong Scrape Config thì cấu hình Job sẽ được load mặc định từ global.
- Với "kube-prometheus-stack" helm chart thì việc khai báo Job trong Scrape Config sẽ được khai báo ở dưới session `additionalScrapeConfigs`
```
    additionalScrapeConfigs:
    - job_name: minio-job
        scrape_interval: 30s
        scrape_timeout: 5s
        metrics_path: /minio/v2/metrics/cluster
        scheme: http
      static_configs:
      - targets: ['minio.monitor.svc.cluster.local:9000']
```

Trong đó: <br>
**job_name:** Tên của Prometheus Job
**scrape_interval:** Thời gian lấy Metrics là mỗi 30s (mỗi 30s, Prometheus gọi tới endpoint của application Minio để lấy Metrics)
**scrape_timeout:** Thời gian timedout là 5s, (sau 5s, Prometheus sẽ tính là timeout nếu không có Metrics được trả về)
**metrics_path:** đường dẫn để lấy Metrics
**target:** endpoint để truy cập tới application Minio

**NOTE: với application cùng namespace với Prometheus thì chỉ cần trỏ đến tên application là đủ (minio), thay vì phải trỏ đến FQDN (minio.monitor.svc.cluster.local)**

Update `kube-prometheus-stack` helm chart để Prometheus apply config mới
```
helm upgrade prometheus-stack -f value-prometheus.yaml kube-prometheus-stack -n monitor
```

- Bài toán đặt ra là khi cần lấy Metric của 100 application, thì tương ứng mỗi application ta phải thêm 1 Job trong Scrape Config. Chưa kể, sau khi thêm Job bạn sẽ phải update Prometheus helm chart để Prometheus apply config mới.
- Như vậy, mỗi khi cấu hình thêm Job mới để pull Metrics từ application mới thì ta phải chỉnh sửa file value.yaml của Prometheus helm chart và update Prometheus helm chart để apply config mới. Đây là lúc Service Monitor phát huy tác dụng

## Service Monitor
- Service Monitor là 1 Object của K8s giúp cho Prometheus có thể tự động detect các Target cần pull Metric. Service Monitor chứa các thông tin về Target giống như Job trong Scrape Config.
## Kiến trúc
![image](https://github.com/user-attachments/assets/3cc31bfe-b29b-4ad1-a7ef-8a7a48a202a9)

## Workflow của Service Monitor
- Prometheus sẽ load config của Job trong Scrape Config và trong Service Monitor object, tất cả các Service Monitor object này sẽ hiển thị ở phần Targets của Prometheus

- Prometheus load config từ Scrape Config nằm trong file config của Prometheus

- Prometheus load config từ Service Monitor object dựa trên 2 tham số:

    - **serviceMonitorNamespaceSelector:** Prometheus sẽ load các Service Monitor object ở namespace nhất định, by default là load từ tất cả namespace

    - **serviceMonitorSelector:** Chọn các Service Monitor sẽ được load để lưu cấu hình Prometheus Job. Ta có thể cấu hình theo nhiều cách khác nhau. Ở đây ta chọn cách cấu hình theo `label` của ServiceMonitor là cách đơn giản nhất

- Config của Service Monitor sẽ có thông tin Label của nó (dùng để Prometheus select) và các thông tin về Target cần monitor (giống các thông tin khai báo cho Prometheus Job trong Scrape Config)

- Như vậy, ta cần cấu hình lại Prometheus để update lại cấu hình Service Monitor selector. Ở đây ta sẽ select từ tất cả namespace, và sẽ filter các Service Monitor có Label là `app.kubernetes.io/instance` và value của Label này là dongna-service-monitor.

- Ta sẽ sửa lại tham số "serviceMonitorSelector:" trong file value.yaml của kube-prometheus-stack (file values-prometheus.yaml) như sau:
```
    serviceMonitorSelector:
      matchExpressions:
      - key: app.kubernetes.io/instance
        operator: In
        values:
          - dongna-service-monitor
          - nginx-ingress-controller          
          - prometheus-stack
```

- Update helm chart để Prometheus apply config mới
```
kubectl upgrade prometheus-stack -f values-prometheus.yaml kube-prometheus-stack -n monitor
```

- Như vậy, các Service Monitor ở các namespace có Label là `app.kubernetes.io/instance` và có 1 trong 3 value này(dongna-service-monitor/nginx-ingress-controller/prometheus-stack) thì sẽ được apply vào Prometheus config.
- Đến đây thì việc monitor thêm 100 application mới đã không còn là vấn đề nữa. Ta chỉ cần gán 1 Label cho toàn bộ các Service Monitor mới match với `serviceMonitorSelector` trong Prometheus config là Prometheus sẽ tự động load config của Service Monitor và không cần phải update lại Prometheus helm chart.
- Với cách tiếp cận này, mỗi application cần monitor bạn sẽ tạo một yaml file cho Service Monitor để tiện lợi cho việc quản lý, cập nhật và tái sử dụng sau này.

- Config mẫu của Service Monitor để giám sát application Minio khá giống với cấu hình Scrape Config, ta tạo file serviceMonitor-minio.yaml với nội dung như sau:
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: minio-monitor
  # Change this to the namespace the Prometheus instance is running in
  namespace: monitor
  labels:
    app.kubernetes.io/instance: dongna-service-monitor
spec:
  endpoints:
  - port: 9000
    interval: 15s
    scheme: http
    metrics_path: /minio/v2/metrics/cluster
  namespaceSelector:
    matchNames:
    - monitor
  selector:
    matchLabels:
      app.kubernetes.io/instance: minio
```
- Trong đó:
    - **namespace:** namespace mà Prometheus đang chạy
    - **app.kubernetes.io/instance: dongna-service-monitor** match với 1 trong 3 serviceMonitorSelector đã khai báo trong Prometheus config
    - **spec:** thông tin Target

- Tạo Service Monitor từ file .yaml
```
kubectl apply -f serviceMonitor-minio.yaml
```

- Output
```
NAME                                 AGE
fluentd                              6d17h
longhorn-prometheus-servicemonitor   21d
minio-monitor                        21d
redis-cluster                        18d
```

- Object Service Monitor tên `minio-monitor` đã được tạo trong `monitor`, và gán Label "app.kubernetes.io/instance: dongna-service-monitor" match với serviceMonitorSelector trong Prometheus config nên Prometheus sẽ load config của Service Monitor này.

Trong Service Monitor object này sẽ có session `spec` chính là thông tin về Target cần monitor, gồm endpoints (port/metric_path), namespaceSelector và selector

Chi tiết hơn, ta có thể get application Minio ở namespace `monitor` với label là `app.kubernetes.io/instance=minio`

```
kubectl get service -n monitor -l "app.kubernetes.io/instance=minio"
```

**NOTE:** Target có 2 loại: Target hỗ trợ endpoint để pull Metric hoặc Target cần dùng Exporter để push Metrics.

- Trong trường hợp này Minio hỗ trợ endpoint để pull Metric nên Prometheus sẽ pull Metrics bằng cách trỏ tới endpoint mà Minio cung cấp.

- Trường hợp application không hỗ trợ endpoint để pull Metric mà phải dùng Exporter để push Metrics thì ta sẽ cần cấu hình phần endpoints và selector trỏ tới Exporter.

## Cài đặt application hỗ trợ sẵn cả Endpoint Metric và Service Monitor (Redis)
- Helm chart hỗ có nhiều open source application hỗ trợ cả Endpoint Metric và template để tạo Service Monitor + Prometheus Rule.
- Ta sẽ cài thử Redis bằng Helm.

```
mkdir -p /home/sysadmin/open-sources
cd /home/sysadmin/open-sources
mkdir redis
cd redis
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo redis
helm pull bitnami/redis-cluster --version 7.5.1
tar -xzf redis-cluster-7.5.1.tgz
cp redis-cluster/values.yaml values-redis.yaml
vim values-redis.yaml
```

- Cấu hình file `value-redis.yaml`
```
usePassword: false
redis: "30101"
type: NodePort
storageClass: "longhorn-storage-delete"
size: 1Gi

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: "monitor"
    labels:
      app.kubernetes.io/instance: dongna-service-monitor
```

- Cài Redis helm chart lên namespace `prod`
```
helm install redis-cluster -f values-redis.yaml -n prod
```

- Kiểm tra lại cài đặt
```
kubectl get pod -n prod -l "app.kubernetes.io/instance=redis-cluster"
kubectl get servicemonitors.monitoring.coreos.com redis-cluster -n monitor
```
- Output

- Ta đã có một cụm Redis Cluster tên `redis-cluster` gồm 6 pods, 3 services trong namespace `prod` và 1 Service Monitor object là `redis-cluster` được tạo trong namespace `monitor` (theo đúng cấu hình ở file values-redis.yaml)
- Kiểm ra xem Service Monitor `redis-cluster` có gì:
```
kubectl describe describe servicemonitors.monitoring.coreos.com redis-cluster -n monitor
```
Output
```

```

Trong đó:<br>
**Labels: app.kubernetes.io/instance=dongna-service-monitor:** Label để Prometheus lựa chọn load config từ Service Monitor nào. Mỗi Service Monitor mới ta sẽ đều thêm label này vào để Prometheus tự động load Service Monitor config.
**Spec:** Spec của Target cần monitor. Ở đây nó tự hiểu cần trỏ đến application ở namespace `prod` mà có gán các Label như bên dưới, ở port tên là metrics.
```

Spec:
  Endpoints:
    Port:  metrics
  Namespace Selector:
    Match Names:
      prod
  Selector:
    Match Labels:
      app.kubernetes.io/component:  metrics
      app.kubernetes.io/instance:   redis-cluster
      app.kubernetes.io/name:       redis-cluster
```
- Verify lại thông tin application được config theo spec trên bằng lệnh:
```
kubectl get service -l "app.kubernetes.io/component=metrics" -l "app.kubernetes.io/instance=redis-cluster" -l "app.kubernetes.io/name=redis-cluster" -n prod
```

- Truy cập Prometheus UI xem application `redis-cluster` đã được hiển thị trong phần Targets hay chưa:


- Tiếp theo, ta import dashboard cho Redis lên Grafana



- Tương tự như Redis, khi cài đặt NGINX-Ingress Controller thì ta có thể enable endpoint Metrics và Service Monitor (tuỳ chỉnh trong file value.yaml) sau đó cũng tạo dashboard trên Grafana để hiển thị Metrics

**Bonus:
- Nếu sử dụng Longhorn storage cho K8s, có thể sẽ gặp phải issue là cấu hình Scrape Config xong thì Prometheus sẽ chỉ lấy được Metrics của 1 node (trong 3 node mà ta config cho longhorn).<br>
- Muốn lấy Metrics của cả Longhorn cluster thì buộc phải sử dụng Service Monitor để Prometheus pull Metrics đầy đủ từ tất cả các longhorn node**

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: longhorn-prometheus-servicemonitor
  namespace: monitoring
  labels:
    name: longhorn-prometheus-servicemonitor
    release:  prometheus-grafana-stack
    k8s-app: longhorn
    app: kube-prometheus-stack
    app.kubernetes.io/instance: dongna-service-monitor
spec:
  selector:
    matchLabels:
      app: longhorn-manager
  namespaceSelector:
    matchNames:
    - storage
  endpoints:
  - port: manager  
```

- Sau đó tìm dashboard cho Grafana (keyword: longhorn example v1.1.0) để hiển thị thông tin lên dashboard:

